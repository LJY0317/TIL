# CLI (Command Line Interface)란?
  명령어를 통해 사용자와 컴퓨터가 사용하는 방식

CLI에서 '.'(점)의 역할
. 현재 디렉토리
.. 현재의 상위 디렉토리 (부모 폴더)

CLI 기초 문법
  touch: 파일의 접근(access) 및 수정(modification) 시간을 현재 시각으로 갱신하거나, 파일이 없으면 빈 파일을 생성함
  mkdir (make directory): 새로운 디렉터리(폴더)를 생성함
  ls (list): 현재 디렉터리의 파일과 폴더 목록을 나열함
  cd (change directory): 현재 작업 디렉터리를 변경함
  start: (Windows) 새 프로세스나 파일, URL 등을 새 창에서 실행함
    MacOS에서는 open을 사용
  rm (remove): 파일을 삭제함
    rm -r (remove recursively): 디렉터리와 그 안의 모든 파일 및 하위 디렉터리를 재귀적으로 삭제함
    rm -rf (remove recursively + force): 사용자 확인 없이 강제로 재귀 삭제함
      쓰기 금지된 파일도 확인 없이 바로 삭제
      존재하지 않는 파일을 지정해도 에러 없이 무시
  pwd (print working directory): 현재 작업 디렉터리의 전체 경로를 출력함
  
루트 디렉토리: /
홈 디렉토리: ~

cd를 이용한 폴더 이동 예시 (폴더 구조는 Desktop/a/b/c 수준이라고 가정):
  cd ..
    : 상위 폴더로 이동
  cd b/c
    : 현재 위치가 .../a일 때 사용하면, .../a/b/c로 한번에 이동!
  cd ~/Desktop/a
    : 절대 경로로 a 폴더까지 한번에 이동!
