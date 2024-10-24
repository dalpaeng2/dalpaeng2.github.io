---
title: Bash Script 오류 처리를 위한 set 설정
date: 2024-10-24T18:59:28+09:00
tags: [shell]
---

셸 스크립트에서 오류 처리는 스크립트의 안정성을 높이는 중요한 요소입니다. 이 포스트에서는 set 명령어와 함께 사용하는 다양한 옵션(e, u, o, E)에 대해 알아보겠습니다.

## set -e

`set -e` 옵션은 스크립트 내에서 명령이 실패할 경우(즉, 종료 코드가 0이 아닌 경우) 즉시 스크립트를 종료하도록 설정합니다. 이 옵션을 사용하면 오류 발생 시 나머지 명령이 실행되는 것을 방지하여, 문제를 조기에 발견할 수 있습니다.

```bash
#!/bin/bash
set -e

echo "Starting the script..."
cd /nonexistent_directory  # 오류 발생
echo "This line will not be executed."  # 이 줄은 실행되지 않음
```

위 예제에서 cd 명령이 실패하면 스크립트가 즉시 종료됩니다.

## set -u

`set -u` 옵션은 정의되지 않은 변수를 사용하려 할 때 오류를 발생시킵니다. 이는 변수를 사용하기 전에 반드시 정의되었는지 확인하는 데 유용합니다. 이를 통해 오타나 의도치 않은 변수를 사용하는 것을 방지할 수 있습니다.

```bash
#!/bin/bash
set -u

echo "Starting the script..."
echo "Value: $undefined_variable"  # 오류 발생
```

이 스크립트는 undefined_variable이 정의되지 않았기 때문에 오류를 발생시키고 종료됩니다.

## set -o pipefail

`set -o pipefail` 옵션은 파이프에서 사용하는 명령 중 하나라도 실패할 경우 전체 파이프의 결과가 실패로 간주되도록 설정합니다. 기본적으로 파이프의 마지막 명령의 종료 코드만 반환되므로, 이 옵션을 사용하면 중간에 있는 명령의 실패를 감지할 수 있습니다.

```bash
#!/bin/bash
set -o pipefail

echo "Starting the script..."
false | echo "This line will execute."  # 오류 발생, 전체 파이프 실패
echo "This line will not be executed."
```

이 경우 `false` 명령이 실패하더라도, 기본적으로는 echo가 실행되므로 전체 파이프가 성공으로 간주될 수 있습니다. set -o pipefail을 사용하면 이 오류를 감지할 수 있습니다.

## set -E

`set -E` 옵션은 함수 내에서 발생한 오류도 trap에 의해 처리될 수 있도록 합니다. 기본적으로, 함수 내에서 발생한 오류는 trap에 의해 처리되지 않을 수 있지만, 이 옵션을 사용하면 함수 내의 오류도 감지할 수 있습니다.

```bash
#!/bin/bash
set -Ee

handle_error() {
    echo "An error occurred in the function."
}

trap handle_error ERR

function_with_error() {
    echo "Executing a function..."
    false  # 오류 발생
}

function_with_error  # 이 줄이 오류를 발생시킴
echo "This line will be executed."  # 이 줄은 계속 실행됨
```

이 경우, function_with_error에서 오류가 발생하면 trap에 의해 handle_error 함수가 호출되지만, set -E만 사용한 경우 스크립트가 종료되지 않고 다음 줄도 실행됩니다.

이와 같이 set 명령어의 다양한 옵션을 사용하여 셸 스크립트의 오류 처리를 더욱 안전하고 효율적으로 할 수 있습니다. 상황에 맞는 적절한 옵션 조합을 선택하여 스크립트를 작성하는 것이 중요합니다.
