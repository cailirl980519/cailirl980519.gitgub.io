---
title: 顏色選擇器 Color Dialog | Excel VBA
date: 2021-12-17 16:42:46
categories: Excel VBA
tags:
- Excel
- VBA
---


### 使用VBA開啟顏色選擇器

![Imgur](https://imgur.com/C11DgCM.gif)

#### STEP 1 新增模組

於VBA Project點擊右鍵插入模組並命名為ColorDialog
> 模組為全域性質，在活頁簿中的所有工作表皆可呼叫

![vba-color-dialog](https://imgur.com/4v5uq1d.png)

#### STEP 2 新增function `Show()`

新增完成後即可在左側模組的資料夾下看到，點擊兩下即可叫出視窗編輯，接著我們就可以在ColorDialog模組下加入`Show()`來開啟顏色選擇器
``` vb
'開啟顏色選擇對話框
Public Function Show() As Long '宣告回傳選中顏色值的型態
    Dim FullColorCode As Long
    '如果有選擇並按確認
    'Application.Dialogs(xlDialogEditColor).Show(Alpha, R, G, B) <-- 初始顏色
    If Application.Dialogs(xlDialogEditColor).Show(1, 84, 96, 164) = True Then
        FullColorCode = ActiveWorkbook.Colors(1)
    End If
    Show = FullColorCode '回傳選中的顏色
End Function
```
<img alt="vba-color-dialog" src="https://imgur.com/sSBjKPU.png" style="width: 80%">


完成後便可以按下執行測試，就可以發現已經可以成功叫出選擇顏色的對話框了！
<img alt="vba-color-dialog" src="https://imgur.com/iAaPvSP.png" style="width: 80%">

#### STEP 3 關聯至工作表

於巨集編輯器左側展開Microsoft Excel物件，打開要關聯的工作表，新增一個`Worksheet_BeforeDoubleClick`，當點擊兩下儲存格時叫出顏色選擇器

``` vb
Private Sub Worksheet_BeforeDoubleClick(ByVal Target As Range, Cancel As Boolean)
    Dim color As Long '顏色
    Dim R, G, B As Integer 'RGB值
    Dim row As Integer '列數
    Dim cell As Range
    '呼叫Module ColorDialog中的Funciton Show
    color = ColorDialog.Show
    '將color轉為RGB
    R = color Mod 256
    G = (color \ 256) Mod 256
    B = color \ 65536
    '獲取點擊列數
    row = Target.row
    '將儲存格設為點擊列數的B欄
    Set cell = Cells(row, 2)
    '設置前一個儲存格背景色
    cell.Previous.Interior.color = color
    '將轉換成HexColor的值放進儲存格
    cell.value = Application.Dec2Hex(R, 2) + Application.Dec2Hex(G, 2) + Application.Dec2Hex(B, 2)
    '將轉換成RGB的值放進右邊儲存格
    cell.Next.value = R & ", " & G & ", " & B
    '設置右側儲存格邊框
    With cell.Next.Borders(xlEdgeRight)
        .LineStyle = xlContinous '邊框樣式
        .Weight = xlThick '邊框粗細
        .color = color '邊框顏色
    End With
End Sub
```

<img alt="vba-color-dialog" src="https://imgur.com/NKXz86U.png" style="width: 80%">