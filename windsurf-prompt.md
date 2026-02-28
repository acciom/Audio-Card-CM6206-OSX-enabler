Automatically build a 64-bit binary from the existing CM6206 OSX enabler source, install it, and create a macOS LaunchDaemon that automatically runs the enabler when the CM6206 USB audio device is connected. All of this must work on macOS Tahoe (macOS 26) on Apple Silicon (M1) without needing an Apple Developer Account or building a kernel extension.

📂 Source Code

Repository to use:
👉 https://github.com/mprinc/Audio-Card-CM6206-OSX-enabler

This source contains the original “enabler” which sends USB commands to enable analog and SPDIF outputs on CM6206 devices.

🧠 Step-by-Step Detailed Instructions
✅ 1) Clone the Source
git clone https://github.com/mprinc/Audio-Card-CM6206-OSX-enabler.git
cd Audio-Card-CM6206-OSX-enabler

WindSurf Task:

Fetch the repo

Enter the directory

✅ 2) Open and Configure the Xcode Project

Open CM6206Init.xcodeproj in Xcode and update:

Build Settings:

Architectures:

arm64
x86_64 (optional if building for Intel)

Remove any 32-bit (i386) targets.

Base SDK: macOS 26

Ensure it will produce a universal 64-bit arm64 (M1) binary.

WindSurf Task:

Update project settings

Remove 32-bit targeting

✅ 3) Patch USB IOKit Code if Needed

If any deprecated IOKit or USB APIs cause build errors:

Replace outdated calls with current macOS Tahoe USB APIs.

Ensure the original logic (USB control requests to enable SPDIF) remains intact.

WindSurf Task:

Apply patches to modernize USB APIs

Preserve SPDIF enable logic

✅ 4) Build the 64-bit Binary
# In Terminal
xcodebuild -configuration Release

Validate the output is arm64:

lipo -info ./build/Release/CM6206Init

WindSurf Task:

Compile the project

Validate the binary’s architecture

✅ 5) Install the Binary

Copy the new binary to /usr/local/bin:

sudo cp ./build/Release/CM6206Init /usr/local/bin/
sudo chmod +x /usr/local/bin/CM6206Init

WindSurf Task:

Install the binary to a system path

Set executable permissions

✅ 6) Test the Binary

Manually run:

sudo /usr/local/bin/CM6206Init

If successful:

The SPDIF/TOSLink indicator on the CM6206 device should light up

macOS should now recognize any outputs enabled by the device

This verifies that the enabler code is working and talking to the USB device.

WindSurf Task:

Run the binary manually

Check hardware SPDIF indicator

🛠 Auto-Run on Device Connect (LaunchDaemon)

launchd can run things automatically on OS events, including USB device attach events. Property lists can specify com.apple.iokit.matching with com.apple.device-attach filters that match USB vendor/product IDs — so that when your CM6206 device is plugged in, the enabler runs automatically.

📄 LaunchDaemon Template

Replace 0x0D8C and 0x0102 with the Vendor ID and Product ID of your CM6206 sound device (hex). Windsurf should auto-fill these with the correct values.

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple Computer//DTD PLIST 1.0//EN"
 "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>

    <!-- Label for the launch daemon -->
    <key>Label</key>
    <string>com.user.cm6206enabler</string>

    <!-- Program to run -->
    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/bin/CM6206Init</string>
    </array>

    <!-- Run when the USB device with matching vendor/product ID is connected -->
    <key>LaunchEvents</key>
    <dict>
        <key>com.apple.iokit.matching</key>
        <dict>
            <key>com.apple.device-attach</key>
            <dict>
                <!-- Apple USB Host class -->
                <key>IOProviderClass</key>
                <string>IOUSBHostDevice</string>

                <!-- Match your device -->
                <key>idVendor</key>
                <integer>0x0D8C</integer>
                <key>idProduct</key>
                <integer>0x0102</key>

                <!-- Ensure launchd treats this as a match event -->
                <key>IOMatchLaunchStream</key>
                <true/>
            </dict>
        </dict>
    </dict>

    <!-- Run as root -->
    <key>UserName</key>
    <string>root</string>

    <!-- Prevent automatic respawn -->
    <key>KeepAlive</key>
    <false/>

</dict>
</plist>

WindSurf Task:

Generate this .plist with correct IDs

Save to:
/Library/LaunchDaemons/com.user.cm6206enabler.plist

📌 Load the LaunchDaemon

Once the plist is in place:

sudo launchctl bootstrap system /Library/LaunchDaemons/com.user.cm6206enabler.plist

WindSurf Task:

Load the daemon using launchctl

Ensure it triggers when the device connects

📌 Key Notes & Tips

✔ The template uses com.apple.iokit.matching with a device-attach matching dictionary — this triggers the job when the specific USB device appears.
✔ If macOS doesn’t natively support this MatchLaunch stream key, fallback options include a daemon script that polls USB devices (but use this only if events don’t fire).
✔ The approach avoids kernel drivers — everything runs in user space.
