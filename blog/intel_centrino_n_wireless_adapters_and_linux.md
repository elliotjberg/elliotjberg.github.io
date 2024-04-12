# The problem
I've had a bunch of issues with these, specifically with the Wireless N 1030 [Rainbow Peak] in my Asus U36SG. When it's happy, it's a really quick card operating at N - but this is a combo WiFi and Bluetooth card, and I use the Bluetooth huge amounts, for example to stream music to a stereo or headphones. I'd noticed more and more that when using both the WiFi and Bluetooth at the same time, the WiFi becomes unstable - running 'iwconfig' will often report speeds of 1Mb/s (sometimes randomly jumping to 6Mb/s), and using anything on the wireless feels horrifically slow - in fact most web pages will just time out and fail to load.

# The interim fix
I tried the general suggestion of disabling N on the card, and this did indeed make the WiFi and Bluetooth work together, but of course I was limited to G speeds (54Mb/s), which is actually really slow now (when your internet is 50Mb/s, and you have a lot of services on the LAN that can operate at much higher speeds - e.g. a NAS). I lived with this for a while, and took to the forums to find a better answer.

# The much more helpful fix
Thanks to someone on the [Ubuntu Forums](http://ubuntuforums.org/showthread.php?t=2268355), I now have fully functional wireless N working happily with my Bluetooth as well - and the fix was actually really simple. I've made both changes suggested in that forum thread, in /etc/modprobe.d/iwlwifi.conf;

    options iwlwifi 11n_disable=8
    options iwlwifi bt_coex_active=N

The first option enables aggregated TX (apparently), see [this launchpad bug](https://bugs.launchpad.net/ubuntu/+source/linux/+bug/1319630) for more info.

The second (to the best of my understanding) disables a special bit of functionality that was designed to make combo BT and WiFi cards (so most of the Centrino series, among a lot of others, but they're very common) work better - and it doesn't really work. So if it did work, there might be an amazing performance increase (probably mostly for the WiFi), but seeing as it often doesn't work, disabling it can actually improve things a lot.

Note that I'd already disabled power saving on the card, which is documented in many places (I followed the instructions [here](http://syntaxionist.rogerhub.com/intel-centrino-wireless-n-2200-ubuntu-1mbps-workaround.html)).
