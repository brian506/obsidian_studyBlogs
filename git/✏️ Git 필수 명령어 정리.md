

# 브랜치 작업

| __상황__      | __명령어__                  |
| ----------- | ------------------------ |
| 로컬에서 브랜치 생성 | git checkout -b [브랜치 이름] |
| 원격 브랜치 가져오기 | git fetch origin         |
| 브랜치 이동      | git checkout [브랜치 이름]    |

# 최신 master 반영

__상황__ : 가장 최근에 merge 된 master 가져오고 싶을 때

1. `git checkout [로컬 브랜치 이름]`
2. `git fetch origin`
3. `git merge origin/master`

# 협업 시 자주 쓰이는 상황

| __상황__        | __명령어__                         |
| ------------- | ------------------------------- |
| 누가 뭘 커밋했는지 보기 | git log --oneline --graph --all |
| 어떤 작업 중인지 보기  | git status                      |
