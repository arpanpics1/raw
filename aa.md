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







Add-Type -AssemblyName System.Windows.Forms
Add-Type -AssemblyName System.Drawing

# Function to simulate mouse click
Add-Type @"
using System;
using System.Runtime.InteropServices;
public class MouseHelper {
    [DllImport("user32.dll")]
    public static extern void mouse_event(int dwFlags, int dx, int dy, int cButtons, int dwExtraInfo);
    public const int MOUSEEVENTF_LEFTDOWN = 0x02;
    public const int MOUSEEVENTF_LEFTUP = 0x04;
    public const int MOUSEEVENTF_MOVE = 0x0001;
    public static void Click() {
        mouse_event(MOUSEEVENTF_LEFTDOWN, 0, 0, 0, 0);
        mouse_event(MOUSEEVENTF_LEFTUP, 0, 0, 0, 0);
    }
}
"@

Write-Host "Running... Press Ctrl+C to stop."

while ($true) {
    # Get current position
    $p = [System.Windows.Forms.Cursor]::Position
    $x = $p.X
    $y = $p.Y

    # Move in a small square pattern
    [System.Windows.Forms.Cursor]::Position = New-Object System.Drawing.Point(($x + 10), $y)
    Start-Sleep -Milliseconds 500
    [System.Windows.Forms.Cursor]::Position = New-Object System.Drawing.Point(($x + 10), ($y + 10))
    Start-Sleep -Milliseconds 500
    [System.Windows.Forms.Cursor]::Position = New-Object System.Drawing.Point($x, ($y + 10))
    Start-Sleep -Milliseconds 500
    [System.Windows.Forms.Cursor]::Position = New-Object System.Drawing.Point($x, $y)
    Start-Sleep -Milliseconds 500

    # Simulate a left click on current position
    [MouseHelper]::Click()

    # Press Shift key (safe, does nothing visible but counts as activity)
    [System.Windows.Forms.SendKeys]::SendWait("+")

    Write-Host "Activity simulated at: $(Get-Date -Format 'HH:mm:ss')"

    # Wait 3 minutes before next activity
    Start-Sleep -Seconds 180
}
