### 해결 방법

1. **원격 변경사항을 로컬에 통합하기**

   ```bash
   # 1) 원격 커밋 가져오기 + 병합
   git pull origin master

   # 2) 충돌이 발생하면 수정 후
   git add <충돌 해결한 파일>
   git commit

   # 3) 다시 푸시
   git push origin master
   ```

2. **rebase 방식으로 깔끔하게 통합하기**

   ```bash
   git pull --rebase origin master
   # 충돌 해결 후
   git rebase --continue
   git push origin master
   ```

3. **(주의!) 강제 푸시로 덮어쓰기**

   * 로컬 이력이 원격보다 더 옳다고 확신할 때만 사용합니다.

   ```bash
   git push --force-with-lease origin master
   ```
---

### How to fix

1. **Merge remote changes into your local branch**

   ```bash
   # 1) Fetch & merge
   git pull origin master

   # If there are conflicts, resolve them, then:
   git add <resolved files>
   git commit

   # 3) Push again
   git push origin master
   ```

2. **Rebase for a cleaner history**

   ```bash
   git pull --rebase origin master
   # Resolve conflicts, then:
   git rebase --continue
   git push origin master
   ```

3. **(Careful!) Force‑push to overwrite remote**

   * Only if you’re sure your local history should replace the remote’s.

   ```bash
   git push --force-with-lease origin master
   ```
