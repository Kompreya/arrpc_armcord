# arrpc_armcord
A homebrewed fix for getting rich presence working with armcord.
These instructions will help you get a functional arrpc server working, get the ArmCord rpc bridge working and injected, and wrap it up in a single script that ensures rpc works each time you launch ArmCord without any extra steps.

![image](https://github.com/Kompreya/arrpc_armcord/assets/41587427/d7b82f42-6e0c-437d-aec6-73c6af99fa51)

![image](https://github.com/Kompreya/arrpc_armcord/assets/41587427/e001113b-0275-450b-9c24-20d96af6dc1f)

1) Grab the latest dev build of ArmCord from its repo, here:  
   https://github.com/ArmCord/ArmCord
   
   Exact version shouldn't matter. Also, this might work with the flatpak, but I haven't tested it. If you decide to try the flatpak, make sure to configure the discord ipc socket permissions via flatseal.  
   If you don't know what that is, see:  
   https://github.com/flathub/xyz.armcord.ArmCord

2) Grab arrpc from here:
   https://github.com/OpenAsar/arrpc
   
   Clone the repo and install it per the instructions. You don't need to do `node src` just yet - my instructions will help you create a script to automate running the arrpc server. Feel free to run it once to test that it works, however.

3) (Optional) Put your armcord appimage somewhere after building it, such as home/.local/bin/armcord or whichever path you typically put appimages/third party applications. Just take note of its location for later.

4) If you haven't done so already, launch ArmCord and let it set things up, and log in. Otherwise, ensure ArmCord is closed, including checking its not running in your system tray or in the background.

5) Navigate to ArmCord's config folder at `~/.config/ArmCord/plugins/loader/`

6) Locate bridge_mod.js. This came with the arrpc files if you cloned it from its repo, located in the `examples` folder.
It can also be found here:  
https://github.com/OpenAsar/arrpc/tree/main/examples  
Move that file to `~/.config/ArmCord/plugins/loader/dist`

7) In `~/.config/ArmCord/plugins/loader/`. open `manifest.json` in a file editor.  
Around line 26, you should see
```json
"resources": ["dist/bundle.js", "dist/bundle.css"],
```  
Modify it to include the bridge_mod.js you just relocated, like so:
```json
"resources": ["dist/bundle.js", "dist/bundle.css", "dist/bridge_mod.js"],
```
Save the file.

8) Back to the loader folder, open `content.js` in a file editor.  
Here, we are modifying the `content.js` to load the `bridge_mod.js` during ArmCord's usual startup actions, but also ensuring it waits since ArmCord first has to load some things the bridge_mod depends on, hence them timeout.

```js
if (typeof browser === "undefined") {
    var browser = chrome;
}

try {
    const script = document.createElement("script");
    script.src = browser.runtime.getURL("dist/bundle.js");
    // documentElement because we load before body/head are ready
    document.documentElement.appendChild(script);
    const style = document.createElement("link");
    style.type = "text/css";
    style.rel = "stylesheet";
    style.href = browser.runtime.getURL("dist/bundle.css");

    document.documentElement.append(script);

    document.addEventListener(
    "DOMContentLoaded",
    () => {
        // Delay loading bridge_mod.js by 10 seconds (adjust timing as needed)
        setTimeout(() => {
            const customScript = document.createElement("script");
            customScript.src = browser.runtime.getURL("dist/bridge_mod.js");
            document.documentElement.appendChild(customScript);
        }, 10000);
    },
    { once: true }
);
} catch (e) {
    console.error(e);
}
```

Save the file.

9) At this point, if you run the arrpc server (such as with `node src` from the arrpc path as per the arrpc github instructions), and then launch ArmCord, rpc should work.  
Try it yourself. Launch the arrpc server in a terminal, then launch ArmCord. Then, launch a game or program you know has Rich Presence, and check your status. If you see an rpc status, then you're good to go!
Steps 5-8 were aimed at getting the `bridge_mod.js` to automatically run when you open ArmCord, so if the arrpc server is working and a game is successfully connected to the arrpc server, but you don't see the game status in your profile in ArmCord, then check the ArmCord devtools console (`ctrl+shift+I`) for any errors relating to `bridge_mod.js`

10) Assuming arrpc and the bridge_mod are working, the next step is to automate the process of starting the server and armcord. First, make sure ArmCord isn't running, and there's no arrpc server running.

11) Create a new user script in your desired folder location. Name it something like `arrpc.sh`, and add the following:
```sh
#!/bin/bash

export NODE_ENV=production

# Checks if arrpc is running
if pgrep -af "node src/index.js" > /dev/null; then
    echo "arrpc server is already running."
else
    echo "Starting arrpc server..."
    cd /path/to/arrpc/
    nohup /path/to/node/version/bin/node src/index.js > /dev/null 2>&1 &
fi
```
To get the exact path to node (`/path/to/node/version/bin/node`), open console and do `which node`. This will spit out something like `/home/kompreya/.nvm/versions/node/v18.18.2/bin/node
` which is what you put in place of `/path/to/node/version/bin/node`  

This script will first check if an arrpc server is running. If so, then it won't do anything. If not, then it'll silently start your arrpc server.  

Save the script and make it executable.  

12) Test the script. Then, check to see if the arrpc server is running, by doing something like `pgrep -af "node src/index.js"` in console. If the server is running, then you'll get a process ID, the path to `node`, and `src/index.js` in console. If nothing is returned in console, then the server isn't running. If the script appears to work, kill the arrpc server and continue.

13) Create another user script. This will be for launching both the arrpc server and ArmCord at the same time. Name it something like `arrpc_armcord.sh` and add the following:

```sh
#!/bin/bash

# Launch arrpc server
/path/to/arrpc.sh

# Launch ArmCord appimage
echo "Launching ArmCord..."
/path/to/ArmCord.AppImage &
```

If you're using the ArmCord flatpak, then you might use something like `flatpak run com.armcord.ArmCord &` instead of `/path/to/ArmCord.AppImage`

In either case, ensure you use the actual name of the .appimage or flatpak.  

Save the file.

15) Test the script. If it works, it should launch arrpc silently (you won't see any indication it's running other than using the afformentioned methods of verifying its running), then launch ArmCord with our `bridge_mod.js` injection. Launch a game or program you know has RPC support, and verify your discord status to see if it shows RPC info.

16) If everything looks good, then you're done! At this point, you can optionally create a .desktop file pointed at the `arrpc_armcord.sh` script or whatever is convenient for you for your setup.




