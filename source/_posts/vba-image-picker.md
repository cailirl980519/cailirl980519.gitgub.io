---
title: 匯入圖片 Image Picker | Excel VBA
date: 2021-12-20 16:11:11
cover: https://i.imgur.com/uss69V2.png
categories: Excel VBA
tags:
- Excel
- VBA
- Image Picker
---

### 使用VBA開啟匯入圖片對話框

<video autoplay muted loop src="https://imgur.com/HzOPsBL.mp4" type="video/mp4" style="width:100%"></video>

#### STEP 1 新增模組

於VBA Project點擊右鍵插入模組並命名為ImagePicker
> 模組為全域性質，在活頁簿中的所有工作表皆可呼叫

![vba-image-picker](https://imgur.com/4v5uq1d.png)

#### STEP 2 新增function `Show()`

新增完成後即可在左側模組的資料夾下看到，點擊兩下即可叫出視窗編輯，接著我們就可以在ImagePicker模組下加入`Show()`來開啟圖片選擇對話框
> Windows及Mac的文件選擇需要使用不同方式呼叫

``` vb
'開啟圖片選擇對話框
Public Function Show() As String '宣告回傳圖片路徑的型態
    #If Mac Then
        Show = GetFileName_Mac '回傳圖片路徑
    #Else
        Show = GetFileName_Windows '回傳圖片路徑
    #End If
End Function
'Mac選擇照片
Function GetFileName_Mac() As String
    Dim startPath As String
    Dim script As String
    Dim result As String

    On Error Resume Next
    startPath = MacScript("return (path to downloads folder) as String")
    script = "set applescript's text item delimiters to "","" " & _
            vbNewLine & _
            "set theFiles to (choose file of type " & _
            " {""png"",""jpg""}" & _
            "with prompt ""Select an image file"" default location alias """ & startPath & _
            """ multiple selections allowed false) as string" & vbNewLine & _
            "set applescript's text item delimiters to """" " & vbNewLine & _
            "return theFiles"
    result = MacScript(script)
    result = Replace$(result, "Macintosh HD", "", Count:=1)
    result = Replace$(result, ":", "/")
    GetFileName_Mac = result
End Function
'Windows選擇照片
Public Function GetFileName_Windows() As String
    With Application.FileDialog(msoFileDialogFilePicker)
        .AllowMultiSelect = False
        .Title = "Select an image file"
        .Filters.Clear
        .Filters.Add "All Pictures", "*.*"
        .Filters.Add "JPG", "*.JPG"
        .Filters.Add "JPEG File Interchange Format", "*.JPEG"
        .Filters.Add "Portable Network Graphics", "*.PNG"
        If .Show = -1 Then
            GetFileName_Windows = .SelectedItems(1)
        End If
    End With
End Function
```

![vba-image-picker](https://imgur.com/Q0fDp3a.png)

完成後便可以按下執行測試，就可以發現已經可以成功叫出選擇選擇圖片的對話框了～可以看到只能選擇我們允許的副檔名

![vba-image-picker](https://imgur.com/uss69V2.png)

#### STEP 3 關聯至工作表

於巨集編輯器左側展開Microsoft Excel物件，打開要關聯的工作表，新增一個Worksheet_BeforeDoubleClick，當點擊兩下儲存格時叫出圖片選擇器

![vba-image-picker](https://imgur.com/5dbpyEt.png)

``` vb
Private Sub Worksheet_BeforeDoubleClick(ByVal Target As Range, Cancel As Boolean)
    Dim path As String '圖片路徑
    Dim row As Integer '列數
    Dim cell As Range
    '呼叫Module ImagePicker中的Funciton Show
    path = ImagePicker.Show
    '獲取當前點擊列數
    row = Target.row
    Set cell = Cells(row, 1)
    If Not path = "" Then
        '將圖片路徑放入儲存格
        cell.value = path
        '將圖片放進工作表
        Set img = ActiveSheet.Pictures.Insert(path)
        '設置圖片大小
        img.Height = 50
        '設置圖片位置（儲存格內置中）
        img.Left = cell.Next.Left + (cell.Width - img.Width) / 2
        img.Top = cell.Top + (cell.Height - img.Height) / 2
    End If
End Sub
```