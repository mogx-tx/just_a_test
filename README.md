### Github Actions方案

### 需求

根据开发同事的描述，需求分为两种情况：

（1）开发者提交单仓PR

（1）开发者提交多仓PR

一、开发者提交单仓PR的时候，开发者在PR中comment “start build”， 触发CI （CI需要拉取所有仓库，做编译验证等操作）

如

```
xxxx
start build
```

二、开发者提交多仓PR的时候，开发者在issue中 评论关联起多笔PR， 并commit “start build”， 触发CI 。

例如：

```
xxxx
https://github.com/vivoblueos/apps_shell/pull/4
https://github.com/vivoblueos/kernel/pull/2
start build
```

### 方案

（1）在每个仓库中配置PR、ISSUE 监听actions

（2）新建一个协调器仓库， 配置actions 。主要作用如下：

- 监听`repository_dispatch`事件
  
- 解析事件，获取需要构建的PR列表（单仓库构建时只有一个PR，多仓库构建时有多个PR）
  
- 初始化repo， 同步所有仓库
  
- 对于每个PR，检出对应的分支（或应用PR变更）到工作区的相应目录
  
- 执行整个项目构建
  

### 具体实施

（1）新建一个public仓库 `repo-orchestrator`

（2）配置一个组织token，给于repo 权限，名称叫`REPO_ORCHESTRATOR_TOKEN01`

（3）每个仓库的actions

.github/workflows/pr_listener.yaml

```
name: PR Listener
 
on:
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]
 
jobs:
  trigger-orchestrator:
    # 检查：评论包含 "start build"，PR/Issue 开放且无冲突
    if: |
      contains(github.event.comment.body, 'start build') &&
      (
        (github.event.issue.state == 'open' && github.event.issue.active_lock_reason != 'too heated') ||
        (github.event.pull_request.state == 'open' && github.event.pull_request.mergeable)
      )
    runs-on: ubuntu-latest
    steps:
      - name: Determine build type
        id: build-type
        run: |
          # 检查是否包含多个PR链接
          if echo '${{ github.event.comment.body }}' | grep -q 'https://github.com/.*/pull/'; then
            echo "build_type=multi-repo-build" >> $GITHUB_OUTPUT
          else
            echo "build_type=single-repo-build" >> $GITHUB_OUTPUT
          fi
           
          # 获取当前仓库信息
          echo "source_repo=${{ github.repository }}" >> $GITHUB_OUTPUT
          echo "pr_number=${{ github.event.issue.number }}" >> $GITHUB_OUTPUT
          echo "comment_id=${{ github.event.comment.id }}" >> $GITHUB_OUTPUT
          echo "sender=${{ github.event.sender.login }}" >> $GITHUB_OUTPUT
 
      - name: Trigger Orchestrator
        uses: actions/github-script@v6
        with:
          github-token: ${{ secrets.REPO_ORCHESTRATOR_TOKEN01 }}
          script: |
            const buildType = '${{ steps.build-type.outputs.build_type }}';
            const sourceRepo = '${{ steps.build-type.outputs.source_repo }}';
            const prNumber = '${{ steps.build-type.outputs.pr_number }}';
            const commentId = '${{ steps.build-type.outputs.comment_id }}';
            const sender = '${{ steps.build-type.outputs.sender }}';
             
            // 对于多仓库构建，提取所有PR链接
            let additionalPayload = {};
            if (buildType === 'multi-repo-build') {
              const commentBody = context.payload.comment.body;
              const prLinks = commentBody.match(/https:\/\/github.com\/[^\/]+\/[^\/]+\/pull\/\d+/g) || [];
              
              additionalPayload.repos = prLinks.map(link => {
                const parts = link.split('/');
                return {
                  owner: parts[3],
                  repo: parts[4],
                  pr: parseInt(parts[6])
                };
              });
            } else {
              // 单仓库构建：添加当前PR信息
              const [owner, repo] = sourceRepo.split('/');
              additionalPayload.repos = [{
                owner,
                repo,
                pr: parseInt(prNumber)
              }];
            }
            
            // 触发协调器
            github.rest.repos.createDispatchEvent({
              owner: 'vivoblueos',
              repo: 'repo-orchestrator',
              event_type: buildType,
              client_payload: {
                source_repo: sourceRepo,
                pr_number: prNumber,
                comment_id: commentId,
                sender: sender,
                ...additionalPayload
              }
            })
```

repo-orchestrator 的actions

注：CI部分需要补充具体做什么

```
name: Orchestrator CI
 
on:
  repository_dispatch:
    types: [single-repo-build, multi-repo-build]
 
jobs:
  handle-build:
    runs-on: ubuntu-latest
      
    steps:
      - name: Parse request
        id: parse-request
        run: |
          # 统一payload格式
          PAYLOAD='${{ toJSON(github.event.client_payload) }}'
          echo $PAYLOAD
           
          # 确保总是有repos数组
          if ! echo "$PAYLOAD" | jq -e '.repos' > /dev/null; then
            # 旧格式转换
            REPOS=$(jq -n --arg repo "$(echo "$PAYLOAD" | jq -r '.source_repo | split("/")[1]')" \
                         --arg owner "$(echo "$PAYLOAD" | jq -r '.source_repo | split("/")[0]')" \
                         --arg pr "$(echo "$PAYLOAD" | jq -r '.pr_number')" \
                         '{repos: [{owner: $owner, repo: $repo, pr: $pr|tonumber}]}')
            PAYLOAD=$(echo "$PAYLOAD" | jq --argjson repos "$REPOS" '. + $repos')
          fi
          
          echo $PAYLOAD
          echo "$PAYLOAD" > payload.json
          echo "Payload content:"
          cat payload.json
          
          # 验证所有PR状态
          jq -c '.repos[]' payload.json | while read repo_data; do
            owner=$(echo "$repo_data" | jq -r '.owner')
            repo=$(echo "$repo_data" | jq -r '.repo')
            pr=$(echo "$repo_data" | jq -r '.pr')
            
            echo "Validating $owner/$repo PR#$pr"
            
            # 检查PR状态
            PR_JSON=$(curl -s -H "Authorization: Bearer ${{ secrets.REPO_ORCHESTRATOR_TOKEN01 }}" \
              "https://api.github.com/repos/$owner/$repo/pulls/$pr")
            echo $PR_JSON
            
            state=$(echo "$PR_JSON" | jq -r '.state')
            mergeable=$(echo "$PR_JSON" | jq -r '.mergeable')
            
            if [ "$state" != "open" ] || [ "$mergeable" != "true" ]; then
              echo "::error::PR $owner/$repo#$pr is not buildable (state: $state, mergeable: $mergeable)"
              exit 1
            fi
          done
 
      - name: Setup environment
        run: |
          sudo apt-get update
          sudo apt-get install -y repo git-lfs python3-pip
          pip3 install PyGithub
           
          # 配置git
          git config --global user.name "BlueOS CI Bot"
          git config --global user.email "buleos_ci@vivo.com"
          git config --global advice.detachedHead false
 
      - name: Initialize repo workspace
        run: |
          mkdir -p blueos
          cd blueos
          ## 这行具体改一下
          repo init https://github.com/vivoblueos/repo-orchestrator -m manifests/default.xml
          repo sync -j8
 
      - name: Apply PR changes
        run: |
          cd blueos
          repo forall -c 'git checkout main ; git pull'
          
          # 处理每个PR
          jq -c '.repos[]' ../payload.json | while read repo_data; do
            owner=$(echo "$repo_data" | jq -r '.owner')
            repo=$(echo "$repo_data" | jq -r '.repo')
            pr=$(echo "$repo_data" | jq -r '.pr')
            repo_path="$repo"
            
            echo "Processing $repo_path PR#$pr"
            
            # 查找项目路径
            project_path=$(repo list | grep "$repo_path" | awk '{print $1}')
            
            if [ -z "$project_path" ]; then
              echo "::error::Project path not found for $repo_path"
              exit 1
            fi
            
            # 进入项目目录
            cd "$project_path"
            
            # 获取PR详细信息
            PR_JSON=$(curl -s -H "Authorization: Bearer ${{ secrets.REPO_ORCHESTRATOR_TOKEN01 }}" \
              "https://api.github.com/repos/$owner/$repo/pulls/$pr")
              
            PR_HEAD_OWNER=$(echo "$PR_JSON" | jq -r '.head.repo.owner.login')
            PR_HEAD_REPO=$(echo "$PR_JSON" | jq -r '.head.repo.name')
            PR_HEAD_REF=$(echo "$PR_JSON" | jq -r '.head.ref')
            PR_HEAD_SHA=$(echo "$PR_JSON" | jq -r '.head.sha')
            echo "PR head: $PR_HEAD_OWNER/$PR_HEAD_REPO@$PR_HEAD_REF ($PR_HEAD_SHA)"
            
            # 添加远程仓库
            git remote add pr-remote "https://github.com/$PR_HEAD_OWNER/$PR_HEAD_REPO.git" || true
            git fetch pr-remote
            
            # 确保主线最新
            MAIN_BRANCH=$(git symbolic-ref refs/remotes/origin/HEAD | sed 's@^refs/remotes/origin/@@')
            git checkout $MAIN_BRANCH
            git pull origin $MAIN_BRANCH
            
            # 创建并切换到PR分支
            git checkout -b "pr-$pr"
            git merge --no-ff "$PR_HEAD_SHA" -m "Merge PR#$pr for CI testing"
            if [ $? -ne 0 ]; then
              echo "::error::Merge conflict in $project_path for PR $pr"
              exit 1
            fi
            
            echo "Applied PR $pr to $project_path"
            echo "Applied PR $pr from $PR_HEAD_OWNER/$PR_HEAD_REPO"
            
            # 返回到repo根目录
            cd - >/dev/null
          done
          
 
      - name: Run CI validation
        run: |
          cd blueos
          ### 执行CI
          echo "start CI"

         
        # 使用超时防止卡死
        timeout-minutes: 120
 
      - name: Report results
        # env:
        #   GITHUB_TOKEN: ${{ secrets.REPO_ORCHESTRATOR_TOKEN01 }}
        run: |
          cd blueos
          CI_STATUS=$([ $? -eq 0 ] && echo "success" || echo "failure")
           
          jq -c '.repos[]' ../payload.json | while read repo_data; do
            owner=$(echo "$repo_data" | jq -r '.owner')
            repo=$(echo "$repo_data" | jq -r '.repo')
            pr=$(echo "$repo_data" | jq -r '.pr')
            
            # 获取项目路径和当前SHA
            project_path=$(repo list | grep "$repo" | awk '{print $1}')
            cd "$project_path"
            COMMIT_SHA=$(git rev-parse HEAD)
            cd - >/dev/null
            
            # # 发送状态报告
            # curl -s -X POST \
            #   -H "Authorization: Bearer $GITHUB_TOKEN" \
            #   -H "Accept: application/vnd.github.v3+json" \
            #   "https://api.github.com/repos/$owner/$repo/statuses/$COMMIT_SHA" \
            #   -d '{
            #     "state": "'$CI_STATUS'",
            #     "target_url": "'$GITHUB_SERVER_URL'/'$GITHUB_REPOSITORY'/actions/runs/'$GITHUB_RUN_ID'",
            #     "description": "BlueOS CI '${CI_STATUS^}'",
            #     "context": "blueos-ci"
            #   }'
            
            echo "Reported $CI_STATUS status for $owner/$repo PR#$pr"
          done
```
