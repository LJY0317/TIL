# CLI (Command Line Interface)란?
  명령어를 통해 사용자와 컴퓨터가 사용하는 방식

## CLI에서 '.'(점)의 역할
. 현재 디렉토리
.. 현재의 상위 디렉토리 (부모 폴더)

## CLI 기초 문법
  - touch: 파일의 접근(access) 및 수정(modification) 시간을 현재 시각으로 갱신하거나, 파일이 없으면 빈 파일을 생성함
  - mkdir (make directory): 새로운 디렉터리(폴더)를 생성함
  - ls (list): 현재 디렉터리의 파일과 폴더 목록을 나열함
  - cd (change directory): 현재 작업 디렉터리를 변경함
  - start: (Windows) 새 프로세스나 파일, URL 등을 새 창에서 실행함
    MacOS에서는 open을 사용
  - rm (remove): 파일을 삭제함
    rm -r (remove recursively): 디렉터리와 그 안의 모든 파일 및 하위 디렉터리를 재귀적으로 삭제함
    rm -rf (remove recursively + force): 사용자 확인 없이 강제로 재귀 삭제함
      쓰기 금지된 파일도 확인 없이 바로 삭제
      존재하지 않는 파일을 지정해도 에러 없이 무시
  - pwd (print working directory): 현재 작업 디렉터리의 전체 경로를 출력함
  
## 그 외
  루트 디렉토리: /
  
  홈 디렉토리: ~

## cd를 이용한 폴더 이동 예시 (폴더는 Desktop/a/b/c 구조라고 가정):
  - cd ..
    : 상위 폴더로 이동
  - cd b/c
    : 현재 위치가 .../a일 때 사용하면, .../a/b/c로 한번에 이동!
  - cd ~/Desktop/a
    : 절대 경로로 a 폴더까지 한번에 이동!


# Markdown
  Heading:
    문서의 단계별 제목으로 사용
    '#'의 개수에 따라 제목의 수준을 구별

  리스트:
    목록을 표시하기 위해 사용
    '1.', '2.' 등: 순서가 있는 리스트가 표시됨
    '-': 순서가 없는 리스트가 표시
    Code Block & Inline code block: 일반 텍스트와 달리 해당 프로그래밍 언어에 맞춰서 텍스트 스타일을 변환
      개발에서 마크다운을 사용하는 가장 큰 이유
        예시) '''phthon
              print('hello')
              '''
      이미지 첨부 가능

그 외 문법
  - **굵게**
  - *기울임*
  - ~~취소선~~

수평선:
  단락을 구분할 때 사용.
  '- (hypen)' 을 3개 이상 적으면 작동됨
    예시
    ---
