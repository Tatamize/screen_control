# screen_control

## Before using

`sudo apt install wmctrl`

And put the 2 files in systemd/user/ folder at the same position (~/.config/systemd/user/)

Run these

systemctl --user daemon-reload
systemctl --user enable chrome-kiosk-screen1.service
systemctl --user enable chrome-kiosk-screen2.service
systemctl --user start chrome-kiosk-screen1.service
systemctl --user start chrome-kiosk-screen2.service
