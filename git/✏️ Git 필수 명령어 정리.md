# 리포지토리와 인텔리제이 연동
1. `git init`
2. `git remote add origin https://github.com/계정이름/리포지토리이름.git
3. `git add .` 
4. `git commit -m "Initial commit"`


# 브랜치 작업

| __상황__      | __명령어__                  |
| ----------- | ------------------------ |
| 로컬에서 브랜치 생성 | git checkout -b [브랜치 이름] |
| 원격 브랜치 가져오기 | git fetch origin         |
| 브랜치 이동      | git checkout [브랜치 이름]    |

# 최신 master 반영

__상황__ : 가장 최근에 merge 된 master 가져오고 다시 작업 이어나갈때

1. `git checkout [로컬 브랜치 이름]`
2. `git pull origin master`
3. `git checkout [내 브랜치]`
4. `git merge master`

# 협업 시 자주 쓰이는 상황

| __상황__        | __명령어__                         |
| ------------- | ------------------------------- |
| 누가 뭘 커밋했는지 보기 | git log --oneline --graph --all |
| 어떤 작업 중인지 보기  | git status                      |
