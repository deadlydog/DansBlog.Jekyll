---
title: PowerShell Multi-Line Input Box Dialog, Open File Dialog, Folder Browser Dialog, Input Box, and Message Box
date: 2013-05-01T18:01:53-06:00
last_modified_at: 2013-12-05T00:00:00-00:00
permalink: /powershell-multi-line-input-box-dialog-open-file-dialog-folder-browser-dialog-input-box-and-message-box/
categories:
  - PowerShell
tags:
  - directory
  - file
  - GUI
  - input
  - input box
  - message box
  - multi line
  - multiline
  - PowerShell
  - prompt
---

I love PowerShell, and when prompting users for input I often prefer to use GUI controls rather than have them enter everything into the console, as some things like browsing for files or folders or entering multi-line text aren't very pleasing to do directly in the PowerShell prompt window. So I thought I'd share some PowerShell code that I often use for these purposes. Below I give the code for creating each type of GUI control from a function, an example of calling the function, and a screen shot of what the resulting GUI control looks like.

[Download a file containing the code for the functions and examples shown here](/assets/Posts/2014/07/PowerShellGuiFunctions.zip)

## Show a message box

Function:

```powershell
# Show message box popup and return the button clicked by the user.
function Read-MessageBoxDialog([string]$Message, [string]$WindowTitle, [System.Windows.Forms.MessageBoxButtons]$Buttons = [System.Windows.Forms.MessageBoxButtons]::OK, [System.Windows.Forms.MessageBoxIcon]$Icon = [System.Windows.Forms.MessageBoxIcon]::None)
{
    Add-Type -AssemblyName System.Windows.Forms
    return [System.Windows.Forms.MessageBox]::Show($Message, $WindowTitle, $Buttons, $Icon)
}
```

Example:

```powershell
$buttonClicked = Read-MessageBoxDialog -Message "Please press the OK button." -WindowTitle "Message Box Example" -Buttons OKCancel -Icon Exclamation
if ($buttonClicked -eq "OK") { Write-Host "Thanks for pressing OK" }
else { Write-Host "You clicked $buttonClicked" }
```

![Message Box Example](/assets/Posts/2014/07/Message-Box-Example.png)

## Prompt for single-line user input

Function:

```powershell
# Show input box popup and return the value entered by the user.
function Read-InputBoxDialog([string]$Message, [string]$WindowTitle, [string]$DefaultText)
{
    Add-Type -AssemblyName Microsoft.VisualBasic
    return [Microsoft.VisualBasic.Interaction]::InputBox($Message, $WindowTitle, $DefaultText)
}
```

Example:

```powershell
$textEntered = Read-InputBoxDialog -Message "Please enter the word 'Banana'" -WindowTitle "Input Box Example" -DefaultText "Apple"
if ($textEntered -eq $null) { Write-Host "You clicked Cancel" }
elseif ($textEntered -eq "Banana") { Write-Host "Thanks for typing Banana" }
else { Write-Host "You entered $textEntered" }
```

![Input Box Example](/assets/Posts/2014/07/Input-Box-Example.png)

## Prompt for a file

This is based on [a post the Scripting Guy made](http://blogs.technet.com/b/heyscriptingguy/archive/2009/09/01/hey-scripting-guy-september-1.aspx).

Function:

```powershell
# Show an Open File Dialog and return the file selected by the user.
function Read-OpenFileDialog([string]$WindowTitle, [string]$InitialDirectory, [string]$Filter = "All files (*.*)|*.*", [switch]$AllowMultiSelect)
{
    Add-Type -AssemblyName System.Windows.Forms
    $openFileDialog = New-Object System.Windows.Forms.OpenFileDialog
    $openFileDialog.Title = $WindowTitle
    if (![string]::IsNullOrWhiteSpace($InitialDirectory)) { $openFileDialog.InitialDirectory = $InitialDirectory }
    $openFileDialog.Filter = $Filter
    if ($AllowMultiSelect) { $openFileDialog.MultiSelect = $true }
    $openFileDialog.ShowHelp = $true    # Without this line the ShowDialog() function may hang depending on system configuration and running from console vs. ISE.
    $openFileDialog.ShowDialog() > $null
    if ($AllowMultiSelect) { return $openFileDialog.Filenames } else { return $openFileDialog.Filename }
}
```

Example:

```powershell
$filePath = Read-OpenFileDialog -WindowTitle "Select Text File Example" -InitialDirectory 'C:\' -Filter "Text files (*.txt)|*.txt"
if (![string]::IsNullOrEmpty($filePath)) { Write-Host "You selected the file: $filePath" }
else { "You did not select a file." }
```

![Select Text File Example](/assets/Posts/2014/07/Select-Text-File-Example.png)

## Prompt for a directory

This is based on [this post](http://forums.anandtech.com/showthread.php?t=2314443), as using `System.Windows.Forms.FolderBrowserDialog` may hang depending on system configuration and running from the console vs. PS ISE.

Function:

```powershell
# Show an Open Folder Dialog and return the directory selected by the user.
function Read-FolderBrowserDialog([string]$Message, [string]$InitialDirectory, [switch]$NoNewFolderButton)
{
    $browseForFolderOptions = 0
    if ($NoNewFolderButton) { $browseForFolderOptions += 512 }

    $app = New-Object -ComObject Shell.Application
    $folder = $app.BrowseForFolder(0, $Message, $browseForFolderOptions, $InitialDirectory)
    if ($folder) { $selectedDirectory = $folder.Self.Path } else { $selectedDirectory = '' }
    [System.Runtime.Interopservices.Marshal]::ReleaseComObject($app) > $null
    return $selectedDirectory
}
```

Example:

```powershell
$directoryPath = Read-FolderBrowserDialog -Message "Please select a directory" -InitialDirectory 'C:\' -NoNewFolderButton
if (![string]::IsNullOrEmpty($directoryPath)) { Write-Host "You selected the directory: $directoryPath" }
else { "You did not select a directory." }
```

![Browse For Folder](/assets/Posts/2014/07/Browse-For-Folder.png)

## Prompt for multi-line user input

This is based on [code shown in this TechNet article](http://technet.microsoft.com/en-us/library/ff730941.aspx).

Function:

```powershell
function Read-MultiLineInputBoxDialog([string]$Message, [string]$WindowTitle, [string]$DefaultText)
{
<#
    .SYNOPSIS
    Prompts the user with a multi-line input box and returns the text they enter, or null if they cancelled the prompt.

    .DESCRIPTION
    Prompts the user with a multi-line input box and returns the text they enter, or null if they cancelled the prompt.

    .PARAMETER Message
    The message to display to the user explaining what text we are asking them to enter.

    .PARAMETER WindowTitle
    The text to display on the prompt window's title.

    .PARAMETER DefaultText
    The default text to show in the input box.

    .EXAMPLE
    $userText = Read-MultiLineInputDialog "Input some text please:" "Get User's Input"

    Shows how to create a simple prompt to get mutli-line input from a user.

    .EXAMPLE
    # Setup the default multi-line address to fill the input box with.
    $defaultAddress = @'
    John Doe
    123 St.
    Some Town, SK, Canada
    A1B 2C3
    '@

    $address = Read-MultiLineInputDialog "Please enter your full address, including name, street, city, and postal code:" "Get User's Address" $defaultAddress
    if ($address -eq $null)
    {
        Write-Error "You pressed the Cancel button on the multi-line input box."
    }

    Prompts the user for their address and stores it in a variable, pre-filling the input box with a default multi-line address.
    If the user pressed the Cancel button an error is written to the console.

    .EXAMPLE
    $inputText = Read-MultiLineInputDialog -Message "If you have a really long message you can break it apart`nover two lines with the powershell newline character:" -WindowTitle "Window Title" -DefaultText "Default text for the input box."

    Shows how to break the second parameter (Message) up onto two lines using the powershell newline character (`n).
    If you break the message up into more than two lines the extra lines will be hidden behind or show ontop of the TextBox.

    .NOTES
    Name: Show-MultiLineInputDialog
    Author: Daniel Schroeder (originally based on the code shown at http://technet.microsoft.com/en-us/library/ff730941.aspx)
    Version: 1.0
#>
    Add-Type -AssemblyName System.Drawing
    Add-Type -AssemblyName System.Windows.Forms

    # Create the Label.
    $label = New-Object System.Windows.Forms.Label
    $label.Location = New-Object System.Drawing.Size(10,10)
    $label.Size = New-Object System.Drawing.Size(280,20)
    $label.AutoSize = $true
    $label.Text = $Message

    # Create the TextBox used to capture the user's text.
    $textBox = New-Object System.Windows.Forms.TextBox
    $textBox.Location = New-Object System.Drawing.Size(10,40)
    $textBox.Size = New-Object System.Drawing.Size(575,200)
    $textBox.AcceptsReturn = $true
    $textBox.AcceptsTab = $false
    $textBox.Multiline = $true
    $textBox.ScrollBars = 'Both'
    $textBox.Text = $DefaultText

    # Create the OK button.
    $okButton = New-Object System.Windows.Forms.Button
    $okButton.Location = New-Object System.Drawing.Size(415,250)
    $okButton.Size = New-Object System.Drawing.Size(75,25)
    $okButton.Text = "OK"
    $okButton.Add_Click({ $form.Tag = $textBox.Text; $form.Close() })

    # Create the Cancel button.
    $cancelButton = New-Object System.Windows.Forms.Button
    $cancelButton.Location = New-Object System.Drawing.Size(510,250)
    $cancelButton.Size = New-Object System.Drawing.Size(75,25)
    $cancelButton.Text = "Cancel"
    $cancelButton.Add_Click({ $form.Tag = $null; $form.Close() })

    # Create the form.
    $form = New-Object System.Windows.Forms.Form
    $form.Text = $WindowTitle
    $form.Size = New-Object System.Drawing.Size(610,320)
    $form.FormBorderStyle = 'FixedSingle'
    $form.StartPosition = "CenterScreen"
    $form.AutoSizeMode = 'GrowAndShrink'
    $form.Topmost = $True
    $form.AcceptButton = $okButton
    $form.CancelButton = $cancelButton
    $form.ShowInTaskbar = $true

    # Add all of the controls to the form.
    $form.Controls.Add($label)
    $form.Controls.Add($textBox)
    $form.Controls.Add($okButton)
    $form.Controls.Add($cancelButton)

    # Initialize and show the form.
    $form.Add_Shown({$form.Activate()})
    $form.ShowDialog() > $null  # Trash the text of the button that was clicked.

    # Return the text that the user entered.
    return $form.Tag
}
```

Example:

```powershell
$multiLineText = Read-MultiLineInputBoxDialog -Message "Please enter some text. It can be multiple lines" -WindowTitle "Multi Line Example" -DefaultText "Enter some text here..."
if ($multiLineText -eq $null) { Write-Host "You clicked Cancel" }
else { Write-Host "You entered the following text: $multiLineText" }
```

![Multi Line Example](/assets/Posts/2014/07/Multi-Line-Example.png)

All of these but the multi-line input box just use existing Windows Forms / Visual Basic controls.

I originally was using the Get verb to prefix the functions, then switched to the Show verb, but after [reading through this page](http://msdn.microsoft.com/en-us/library/windows/desktop/ms714428%28v=vs.85%29.aspx), I decided that the Read verb is probably the most appropriate (and it lines up with the Read-Host cmdlet).

Hopefully you find this useful.

Happy coding!
