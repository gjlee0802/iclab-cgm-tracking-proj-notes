# xDrip 알아보기
## Overview
* xDrip Github - https://github.com/NightscoutFoundation/xDrip
* 지원 CGM (Bluetooth connection)
    * Dexcom G5~G7: 확인됨
    * CareSens Air: 확인됨
    * Freestyle Libre 2: 확인되지 않음

## 사용 Tips
* Dexcom 사용할 경우, Settings에 들어가서 Transmitter ID를 입력해줘야 함.
    * G7의 경우 4자리 숫자에 해당함.


* 왼쪽 사이드 메뉴를 열어  **Settings 탭**을 누르면 상세 설정이 가능함.
    * Hardware Data Source: 
        * 데이터 수집 대상 기기 설정
        * Dexcom Transmitter ID: 
        * Hardware Data Source를 Dexcom으로 설정할 경우 설정할 수 있으며, G7의 경우 위에 언급된 4자리 숫자를 입력하면 됨.
    * Dex Debug Settings:  
        * Hardware Data Source를 Dexcom으로 설정할 경우 설정할 수 있으며, Dexcom에 대한 상세 설정이 가능함.
        * G6/G7/Dex1/One+ Support: Dexcom G5가 아닌 상위 버전 기기라면 체크 필수!
* 오른쪽 상단의 버튼 내 View Events Log를 누르면 상세 로그 확인 가능
    * 동작에 있어서의 문제점이나 이벤트의 메시지를 확인할 수 있음.
    * 실 사용 중에 생기는 문제는 View Events Log를 통해 로그 확인하기.
