# Instagram Non-Followers Finder

A lightweight, client-side JavaScript snippet that runs directly in your browser's developer console to identify Instagram accounts you follow that do not follow you back, and exports the results to a clean JSON file.

## Features

- **Zero External Dependencies:** Runs natively in any modern web browser without installing third-party extensions or libraries.
- **Session-Aware:** Automatically retrieves your active session ID (`ds_user_id`) from your current browser cookies.
- **Rate-Limit Protection:** Includes randomized delays between requests and periodic 10-second cooldowns every 6 requests to prevent temporary account restrictions.
- **Console Summary & Table:** Displays real-time progress logs, total counts, and a formatted `console.table` of non-followers.
- **Automatic JSON Export:** Downloads a `usersNotFollowingBack.json` file containing usernames, full names, verification status, and direct profile URLs.

## How It Works

1. **Session Verification:** The `getCookie` helper extracts `ds_user_id` from `document.cookie` to verify that you are logged into Instagram.
2. **Paginated Data Fetching:** The `fetchAll(type)` async function queries Instagram's internal friendships API (`following` and `followers`) in batches of 100 using `max_id` pagination until all users are retrieved.
3. **Throttling:** To mimic human browsing behavior and avoid triggering spam filters, the script pauses for `1000–1400 ms` after each request and adds an extra `10-second` pause every 6 API calls.
4. **Comparison & Filtering:** Follower IDs are stored in a JavaScript `Set` for fast $O(1)$ lookups, filtering out anyone from your `following` list who appears in your `followers` list.
5. **Output Generation:** Results are printed in the console and packaged into a `Blob` to trigger an automatic download of `usersNotFollowingBack.json`.

## Code

```javascript
(async () => {
  const getCookie = (name) => {
    const parts = `; ${document.cookie}`.split(`; ${name}=`);
    return parts.length === 2 ? parts.pop().split(";").shift() : null;
  };
  const sleep = (ms) => new Promise((r) => setTimeout(r, ms));
 
  const userId = getCookie("ds_user_id");
  if (!userId) {
    console.error("No se encontró la sesión. Inicia sesión en instagram.com y vuelve a intentarlo.");
    return;
  }
 
  const headers = { "X-IG-App-ID": "936619743392459" };
  let requests = 0;
 
  // Descarga una lista completa: "following" (seguidos) o "followers" (seguidores)
  async function fetchAll(type) {
    const users = [];
    let maxId = "";
    do {
      const url =
        `https://www.instagram.com/api/v1/friendships/${userId}/${type}/?count=100` +
        (maxId ? `&max_id=${maxId}` : "");
 
      const res = await fetch(url, { headers, credentials: "include" });
      if (!res.ok) throw new Error(`Error ${res.status} al pedir ${type}`);
 
      const data = await res.json();
      users.push(...data.users);
      maxId = data.next_max_id || "";
 
      console.log(`%c ${type}: ${users.length} cargados...`, "color:#bada55;font-size:14px;");
 
      // Pausas para evitar bloqueos temporales
      await sleep(1000 + Math.floor(Math.random() * 400));
      if (++requests % 6 === 0) {
        console.log("%c Pausa de 10 s para evitar bloqueos...", "color:#FC4119;font-size:14px;");
        await sleep(10000);
      }
    } while (maxId);
    return users;
  }
 
  try {
    const following = await fetchAll("following");
    const followers = await fetchAll("followers");
 
    const followerIds = new Set(followers.map((u) => u.pk || u.id));
    const notFollowingBack = following
      .filter((u) => !followerIds.has(u.pk || u.id))
      .map((u) => ({
        username: u.username,
        full_name: u.full_name,
        is_verified: u.is_verified,
        profile: `https://www.instagram.com/${u.username}/`,
      }));
 
    console.clear();
    console.log(
      `%c Sigues a ${following.length} | Te siguen ${followers.length} | No te siguen de vuelta: ${notFollowingBack.length}`,
      "background:#222;color:#bada55;font-size:18px;"
    );
    console.table(notFollowingBack);
 
    // Descarga el resultado como JSON
    const blob = new Blob([JSON.stringify(notFollowingBack, null, 2)], { type: "application/json" });
    const a = document.createElement("a");
    a.href = URL.createObjectURL(blob);
    a.download = "usersNotFollowingBack.json";
    a.click();
 
    console.log("%c ¡Listo!", "background:#222;color:#bada55;font-size:20px;");
  } catch (err) {
    console.error("Algo falló:", err);
  }
})();
```

## Usage

1. Open [Instagram](https://www.instagram.com/) in your desktop web browser and log into your account.
2. Open the browser's **Developer Tools**:
   - **Windows / Linux:** Press `Ctrl + Shift + I` or `F12`
   - **macOS:** Press `Cmd + Option + I`
3. Navigate to the **Console** tab.
4. Copy the JavaScript code above, paste it into the console, and press `Enter`. *(Note: If your browser blocks pasting into the console, type `allow pasting` and press `Enter` first).*
5. Wait for the script to finish fetching both lists. Once completed, a table will appear in the console and `usersNotFollowingBack.json` will download automatically.

## Output Example

```json
[
  {
    "username": "example_user",
    "full_name": "Example User",
    "is_verified": false,
    "profile": "https://www.instagram.com/example_user/"
  }
]
```

## Disclaimer

This project is intended for personal and educational purposes only. Automating requests on Instagram may go against its Terms of Service. Although this script includes built-in delays to minimize risks, use it responsibly and avoid running it repeatedly in short periods.

## Author

Created by [@alexitotroste2007-afk](https://github.com/alexitotroste2007-afk)
