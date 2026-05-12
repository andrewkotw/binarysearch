# Google Apps Script Documentation

This document explains the Google Apps Script used by `quiz.html` to save quiz results into a Google Sheet.

## Purpose

The script receives quiz result data from the website, creates or finds a sheet named `測驗成績`, and appends one row for each submitted quiz.

It is designed to keep wrong-question tracking based on the original question numbers. Even if the quiz randomizes the question order for students, the submitted `wrongQuestions[].number` should still refer to the original question number from `quiz.html`.

For example:

- Original question 7 appears as the first question on a student's screen.
- The student answers it incorrectly.
- The submitted data records it as question `7`.
- The Google Sheet stores `[7]` in the `錯題題號` column.

## Sheet Name

```javascript
const SHEET_NAME = '測驗成績';
```

The script writes all quiz results to a sheet named `測驗成績`.

If that sheet does not exist, the script creates it automatically.

## Sheet Columns

When the sheet is empty, the script adds this header row:

| Column | Meaning |
|---|---|
| 提交時間 | Time the quiz was submitted |
| 測驗名稱 | Quiz title |
| 姓名 | Student name |
| 班級 | Class number |
| 座號 | Seat number |
| 答對題數 | Number of correct answers |
| 答錯題數 | Number of wrong answers |
| 總題數 | Total number of questions |
| 得分比例 | Score percentage |
| 錯題題號 | Original question numbers answered incorrectly |

## Expected Payload

The website sends the result as a JSON string inside a form field named `payload`.

Example payload:

```json
{
  "quizTitle": "搜尋演算法小測驗",
  "name": "王小明",
  "classInfo": "802 / 13",
  "classNumber": "802",
  "seatNumber": "13",
  "correct": 8,
  "wrong": 2,
  "total": 10,
  "percent": 80,
  "wrongQuestions": [
    {
      "number": 7,
      "displayNumber": 1,
      "question": "二元搜尋中，如果 L > R，通常代表什麼？"
    },
    {
      "number": 10,
      "displayNumber": 4,
      "question": "如果一組資料是 [56, 12, 89, 34, 7]，要直接找某個數字，最安全的搜尋法是什麼？"
    }
  ]
}
```

The important field for difficulty tracking is:

```javascript
wrongQuestions[].number
```

This should be the original question number, not the randomized display number.

## Main Function: `doPost(e)`

```javascript
function doPost(e) {
  try {
    const payloadText = e.parameter.payload;
    if (!payloadText) {
      return jsonOutput({ ok: false, error: 'missing payload' });
    }

    const data = JSON.parse(payloadText);
    const sheet = getOrCreateSheet_();
    const classSeat = parseClassSeat_(data.classInfo || '');
    const classNumber = data.classNumber || classSeat.classNumber;
    const seatNumber = data.seatNumber || classSeat.seatNumber;
    const wrongQuestionNumbers = (data.wrongQuestions || [])
      .map(item => item.number)
      .join(',');
    const wrongQuestionText = `[${wrongQuestionNumbers}]`;

    sheet.appendRow([
      new Date(),
      data.quizTitle || '',
      data.name || '',
      classNumber || '',
      seatNumber || '',
      data.correct ?? '',
      data.wrong ?? '',
      data.total ?? '',
      data.percent ?? '',
      wrongQuestionText
    ]);

    return jsonOutput({ ok: true });
  } catch (error) {
    return jsonOutput({ ok: false, error: String(error) });
  }
}
```

What it does:

1. Reads `payload` from the POST request.
2. Parses it from JSON text into a JavaScript object.
3. Finds or creates the results sheet.
4. Gets class and seat numbers.
5. Extracts original wrong-question numbers from `data.wrongQuestions`.
6. Formats them like `[2,7,10]`.
7. Appends one row to the sheet.
8. Returns a JSON response.

## Class and Seat Parsing

```javascript
function parseClassSeat_(rawValue) {
  const raw = String(rawValue || '').trim();
  const separated = raw.match(/^(\d{3})\D+(\d{1,3})$/);
  const compact = raw.replace(/\D/g, '');
  let classNumber = '';
  let seatNumber = '';

  if (separated) {
    classNumber = separated[1];
    seatNumber = String(Number(separated[2]));
  } else if (compact.length >= 5) {
    classNumber = compact.slice(0, 3);
    seatNumber = String(Number(compact.slice(3)));
  }

  return { classNumber, seatNumber };
}
```

This helper supports formats like:

| Student Input | Parsed Class | Parsed Seat |
|---|---:|---:|
| `80213` | `802` | `13` |
| `802/13` | `802` | `13` |
| `802-13` | `802` | `13` |
| `802013` | `802` | `13` |

## Sheet Setup

```javascript
function getOrCreateSheet_() {
  const spreadsheet = SpreadsheetApp.getActiveSpreadsheet();
  let sheet = spreadsheet.getSheetByName(SHEET_NAME);

  if (!sheet) {
    sheet = spreadsheet.insertSheet(SHEET_NAME);
  }

  if (sheet.getLastRow() === 0) {
    sheet.appendRow([
      '提交時間',
      '測驗名稱',
      '姓名',
      '班級',
      '座號',
      '答對題數',
      '答錯題數',
      '總題數',
      '得分比例',
      '錯題題號'
    ]);
    sheet.setFrozenRows(1);
  }

  return sheet;
}
```

This helper:

- Opens the active spreadsheet.
- Looks for the `測驗成績` sheet.
- Creates the sheet if missing.
- Adds the header row if the sheet is empty.
- Freezes the header row.

## JSON Response Helper

```javascript
function jsonOutput(data) {
  return ContentService
    .createTextOutput(JSON.stringify(data))
    .setMimeType(ContentService.MimeType.JSON);
}
```

This returns a JSON response, such as:

```json
{ "ok": true }
```

or:

```json
{ "ok": false, "error": "missing payload" }
```

## Deployment Steps

1. Open the target Google Sheet.
2. Go to `Extensions > Apps Script`.
3. Paste the Apps Script code into the editor.
4. Save the project.
5. Click `Deploy > New deployment`.
6. Choose `Web app`.
7. Set `Execute as` to `Me`.
8. Set access to the correct audience for your classroom setup.
9. Deploy and authorize the script.
10. Copy the Web App URL.
11. Paste the URL into `GOOGLE_SHEET_WEB_APP_URL` in `quiz.html`.

## Notes for Randomized Questions

The Apps Script does not need to know the randomized display order. It only stores what the website sends.

The website should send:

```javascript
wrongQuestions: [
  { number: 7, displayNumber: 1 },
  { number: 10, displayNumber: 4 }
]
```

The Apps Script stores only:

```text
[7,10]
```

This keeps your spreadsheet useful for tracking which original questions are most difficult across all students.

