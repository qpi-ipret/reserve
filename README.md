# reserve

충곽 스카

## guide

유지보수 시 다음을 확인하시기 바랍니다.

### backend distribution

이 프로그램은 백?엔드로 구글 스프레드시트와 연동된 apps script를 사용합니다. 기본적으로, apps script에서 무언가를 바꾸더라도 프로그램에 적용되지 않습니다. 이는 apps script가 버전이 지정된 배포를 사용해서, apps script에서의 변경 사항이 배포에 영향을 끼치지 않기 때문입니다. apps script에서 바꾼 내용을 적용하고 싶다면 다음과 같은 작업을 진행하시기 바랍니다.

1. 우상단의 파란색 '배포' 버튼을 누른 뒤, '배포 관리'를 선택합니다.
2. 유일하게 보관처리가 되지 않고 '활성'에 있는 배포를 선택합니다.
3. 우상단의 '편집' 버튼(연필 모양 버튼)을 누른 뒤, '버전'을 '새 버전'으로 바꿉니다.
4. 우하단의 '배포' 버튼을 눌러 적용합니다.

### backend source code

프론트엔드 쪽의 소스 코드는 main 브랜치에서 볼 수 있으나, 백엔드 쪽의 소스 코드는 깃허브에서 볼 수 없습니다. 따라서 이곳에 백엔드 소스 코드를 첨부합니다.

#### 로그인 스크립트(AUTH_URL)

```js
function doPost(e) {
  try {
    var body = JSON.parse(e.postData.contents);
    var idToken = body.idToken;

    // Verify token with Google
    var url = "https://oauth2.googleapis.com/tokeninfo?id_token=" + idToken;
    var response = UrlFetchApp.fetch(url);
    var payload = JSON.parse(response.getContentText());
    var date = Date.now()

    return ContentService.createTextOutput(
      JSON.stringify({ success: true, pfp: payload.picture, sub: payload.sub, date: date, auth: HMACSHA256(`${payload.sub}@${date}`,"----(비공개)----") })
    ).setMimeType(ContentService.MimeType.JSON);

  } catch (err) {
    return ContentService.createTextOutput(
      JSON.stringify({ success: false, error: err.message })
    ).setMimeType(ContentService.MimeType.JSON);
  }
}

function HMACSHA256(value, key) {
  // Compute the HMAC-SHA256 signature
  var signatureBytes = Utilities.computeHmacSha256Signature(value, key);
  
  // Convert the signature bytes to a hexadecimal string (or Base64 if needed)
  var hexSignature = signatureBytes.map(function(byte) {
    return ('0' + (byte & 0xFF).toString(16)).slice(-2);
  }).join('');
  console.log(hexSignature);
  return hexSignature;
}
```

#### 스프레드시트 업데이트 스크립트(URL)

```js
function formatDate(d) {
  return `${d.getFullYear()}-${(d.getMonth()+1).toString().padStart(2,'0')}-${d.getDate().toString().padStart(2,'0')}`;
}

function HMACSHA256(value, key) {
  // Compute the HMAC-SHA256 signature
  var signatureBytes = Utilities.computeHmacSha256Signature(value, key);
  
  // Convert the signature bytes to a hexadecimal string (or Base64 if needed)
  var hexSignature = signatureBytes.map(function(byte) {
    return ('0' + (byte & 0xFF).toString(16)).slice(-2);
  }).join('');
  console.log(hexSignature);
  return hexSignature;
}

function doPost(e) {
  try {
    const spreadsheet = SpreadsheetApp.openById('----(비공개)----');  // --(1)
    const sheet = spreadsheet.getSheetByName('----(비공개)----');  // --(2)
    const data = JSON.parse(e.postData.contents);
    const today = formatDate(new Date());
    const timestamp = Date.now();
    
    if (data.auth != HMACSHA256(`${data.sub}@${data.authdate}`, "----(비공개)----")) {
      return ContentService
        .createTextOutput("fail: 로그인 인증에 실패했어요.")
    }
    if (timestamp - data.authdate > 3600000) {
      return ContentService
        .createTextOutput("fail: 로그인한 지 너무 오래 지났어요. 새로고침하고 다시 해 보세요.")
    }

    const startDate = new Date().setHours(13, 20, 0, 0)
    const endDate = new Date().setHours(18, 20, 0, 0)
    const now = new Date()
    if (sheet.getRange("U1").getValue()==1 && !(startDate <= now && now <= endDate)) {
      return ContentService
        .createTextOutput("fail: 오픈 시간이 아니에요.")
    }

    if (data.studentId == "DELETE") {
      const key = data.key;
      const sub = data.sub;
      const range = sheet.getRange(1,1,sheet.getLastRow(),5);
      const value = range.getValues();
      let result = [];
      let success = false;
      for (let i=0; i<value.length; i++) {
        if (value[i][2] != today) continue;
        result.push(value[i]);
        if (value[i][0] != key) continue;
        if (value[i][4] == sub) {
          sheet.getRange(i+1,1,1,5).clearContent();
          result.pop();
          success = true;
        } else {
          return ContentService.createTextOutput(`fail: 예약 시 사용한 구글 계정이 아니에요.`);
        }
      }
      if (success) {
        return ContentService.createTextOutput(`${JSON.stringify(result)}`)
      }
      return ContentService.createTextOutput(`fail: 삭제할 예약이 존재하지 않아요.\n${JSON.stringify(result)}`);
    }
    const rowData = [
      data.studentId,
      data.name,
      today,
      timestamp,
      data.sub
    ];  // --(3)

    const range = sheet.getRange(1,1,sheet.getLastRow(),5);
    const value = range.getValues();
    let result = []
    let exists1 = false
    let exists2 = false
    for (let i=0; i<value.length; i++) {
      if (value[i][2] != today) continue;
      if (value[i][0] == rowData[0]) exists1 = true
      if (value[i][4] == rowData[4]) exists2 = true
      result.push(value[i]);
    }
    if (result.length >= sheet.getRange("Z1").getValue()) {
      return ContentService.createTextOutput(`fail: 오늘 예약이 마감되었어요.\n${JSON.stringify(result)}`)
    }
    if (exists2) {
      return ContentService.createTextOutput(`fail: 이미 사용된 구글 계정이에요.\n${JSON.stringify(result)}`)
    }
    if (exists1) {
      return ContentService.createTextOutput(`fail: 이미 사용된 학번이에요.\n${JSON.stringify(result)}`)
    }

    // Write the form data to the spreadsheet
    const newRow = sheet.getLastRow() + 1;
    sheet.getRange(newRow,1,1,rowData.length).setValues([rowData]);

    result.push(rowData);
    return ContentService
      .createTextOutput(JSON.stringify(result))
  } catch (error) {
    return ContentService
      .createTextOutput(`fail: 에러 발생. ${error}. 개발자에게 문의해 주세요.`)
  }
}

function doGet(e) {
  try {
    const spreadsheet = SpreadsheetApp.openById('----(비공개)----');  // --(1)
    const sheet = spreadsheet.getSheetByName('----(비공개)----');  // --(2)
    const range = sheet.getRange(1,1,sheet.getLastRow(),5);
    const value = range.getValues();
    const today = formatDate(new Date());
    let result = []
    for (let i=0; i<value.length; i++) {
      if (value[i][2] != today) continue;
      result.push(value[i]);
    }
    return ContentService.createTextOutput(JSON.stringify(result))
  } catch (error) {
    return ContentService.createTextOutput(`fail: ${error}`)
  }
}
```

#### 초기화 스크립트(자동 실행 중)

```js
function resetColumnManual() {
  const sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName("----(비공개)----");
  const lastRow = sheet.getLastRow();

  if (lastRow > 0) {
    // A열(1) ~ E열(5)까지 모든 행 초기화
    sheet.getRange(1, 1, lastRow, 5).clearContent();
  }
}


// 매일 지정한 시각(예: 23:40)에 반복 트리거 생성
function createDailyTrigger(hour = 9, minute = 0) {
  // 기존 resetLocations 트리거 삭제
  ScriptApp.getProjectTriggers().forEach(t => {
    if (t.getHandlerFunction() === "resetColumnManual") {
      ScriptApp.deleteTrigger(t);
    }
  });

  // 매일 반복 트리거 (atHour + nearMinute + everyDays)
  ScriptApp.newTrigger("resetColumnManual")
    .timeBased()
    .atHour(hour)          // 0~23
    .nearMinute(minute)    // 0~59 (정확히 그 분 전후 15분 내 실행될 수 있음)
    .everyDays(1)
    .create();

  console.log(`트리거 생성: 매일 ${hour}시 ${minute}분(근처)에 실행`);
}

// 날짜가 바뀌었는데 혹시 트리거가 못 돌았을 때, 첫 편집 시 보정
function maybeDailyReset_() {
  const props = PropertiesService.getScriptProperties();
  const last = props.getProperty("lastReset");
  const today = new Date().toDateString();
  if (last !== today) {
    resetColumnManual();
  }
}


function debugCreateTrigger() {
  // 같은 프로젝트의 resetLocations 트리거 전부 삭제
  ScriptApp.getProjectTriggers().forEach(t => {
    if (t.getHandlerFunction() === 'resetColumnManual') ScriptApp.deleteTrigger(t);
  });

  // 새로 생성
  ScriptApp.newTrigger('resetColumnManual')
    .timeBased()
    .atHour(9)       // 0~23
    .nearMinute(0)   // 0~59 (±15분 오차 범위)
    .everyDays(1)
    .create();

  // 현재 트리거 목록 로깅
  const info = ScriptApp.getProjectTriggers().map(t => ({
    fn: t.getHandlerFunction(),
    type: t.getEventType(),
    src: t.getTriggerSource()
  }));
  Logger.log(JSON.stringify(info));
}
```

#### 수동 초기화 스크립트(디버그용)

```js
function resetColumnManual() {
  const sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName("시트1");
  const lastRow = sheet.getLastRow();

  if (lastRow > 0) {
    // A열(1) ~ E열(5)까지 모든 행 초기화
    sheet.getRange(1, 1, lastRow, 5).clearContent();
  }
}
```

## LICENSE

```text
React
react-dom-client.production.js
react-dom.production.js
react.production.js
scheduler.production.js

Copyright (c) Meta Platforms, Inc. and affiliates.

This source code is licensed under the MIT license.

----------------

MIT License

Copyright (c) Meta Platforms, Inc. and affiliates.

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

```text
A tiny SHA256 implementation (https://github.com/6502/sha256/tree/main)

Copyright 2022 Andrea Griffini

Permission is hereby granted, free of charge, to any person obtaining
a copy of this software and associated documentation files (the
"Software"), to deal in the Software without restriction, including
without limitation the rights to use, copy, modify, merge, publish,
distribute, sublicense, and/or sell copies of the Software, and to
permit persons to whom the Software is furnished to do so, subject to
the following conditions:

The above copyright notice and this permission notice shall be
included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND,
EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND
NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE
LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION
OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION
WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
```

<!-- <br> <br> patafim -->
<!-- 죄송합니다 -->