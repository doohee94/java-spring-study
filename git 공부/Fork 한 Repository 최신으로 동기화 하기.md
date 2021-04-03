## Fork한 repository 최신으로 동기화 하기

### 1. git remote -v

-  현재 연결된 remote 확인 -> 내 repository에 있는 원격이어야함!

### 2. git remote add upstream {원본 repository 주소}

- 동기화 해오고 싶은 원본 repository 를 upstream 이라는 이름으로 추가한다. 

### 3. git fetch upstream 

- 원본 repository의 최신 내용을 가져온다. 

### 4. git checkout {branch}

- 원하는 브랜치로 체크아웃

### 5. git merge upstream/{branch}

- upstream에 원하는 브랜치를 현재 브랜치로 merge

### 6. git push

- 내 repository로 push

  

​	



