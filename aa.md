@echo off
powershell -Command "Add-Type -AssemblyName System.Windows.Forms; while ($true) { $p = [System.Windows.Forms.Cursor]::Position; [System.Windows.Forms.Cursor]::Position = New-Object System.Drawing.Point(($p.X + 10), $p.Y); Start-Sleep -Seconds 30 }"




Add-Type -AssemblyName System.Windows.Forms
while ($true) {
    $p = [System.Windows.Forms.Cursor]::Position
    [System.Windows.Forms.Cursor]::Position = New-Object System.Drawing.Point(($p.X + 10), $p.Y)
    Start-Sleep -Seconds 30
}


@echo off
powershell -ExecutionPolicy Bypass -File "%~dp0move_mouse.ps1"
