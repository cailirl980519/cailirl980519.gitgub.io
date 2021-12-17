---
title: 新手上路 - Excel VBA
date: 2021-12-17 10:23:31
categories: Excel VBA
tags:
- Excel
- VBA
---

### 開啟VBA
點擊工作表右鍵 > 檢視程式碼，即可看到編輯VBA的視窗
![vba-begin](https://imgur.com/1s7J9Ql.png)
<img alt="vba-begin" src="https://imgur.com/5YSI1h8.png" style="width: 80%;">

### 開啟開發人員工作列
- Windows: 檔案 > 選項 > 自訂功能區 > 勾選開發人員
- Mac: Excel > 喜好設定 > 功能區和工具列 > 勾選開發人員
![vba-begin](https://imgur.com/VTBQgK7.png)

### 使用按鈕複製選取儲存格文字

#### STEP 1: 插入按鈕
- Windows: 開發人員 > 插入 > 表單控制項中的按鈕
- Mac: 開發人員 > 按鈕
接著就在工作表中放置按鈕
![vba-begin](https://imgur.com/Jnyyw9z.png)
> 若要移動按鈕位置或改變大小，Windows需點擊開發人員中的設計模式; Mac則需右鍵點擊按鈕

#### STEP 2: 指定巨集
新增一個巨集，當按鈕按下後會呼叫
![vba-begin](https://imgur.com/niJHjvK.png)
新增完成即會跳到巨集編輯器
<img alt="vba-begin" src="https://imgur.com/bCf7vGl.png" style="width: 80%;">

#### STEP3: 編輯巨集功能
接著我們就可以開始編寫複製功能
- Selection.Address: 當前選取的儲存格地址
- MsgBox: 提示框
``` vbscript
Sub Btn()
    '選取儲存格的數量
    If Range(Selection.Address).Count = 1 Then
        '將文字丟進剪貼簿
        Dim obj As New DataObject
        obj.SetText Range(Selection.Address).value
        obj.PutInClipboard
        MsgBox "已複製：" & obj.GetText
    Else
        MsgBox "無法複製多個儲存格"
    End If
End Sub
```

#### STEP4: 點擊按鈕
寫完程式碼後就可以開始測試效果了～
![vba-begin](https://imgur.com/LNDpGpA.gif)

### 儲存
儲存有巨集的Excel時必須存成**啟用巨集的活頁簿（`.*xlsm`）**，若存為一般的`.*xls`是無法啟用巨集的哦～