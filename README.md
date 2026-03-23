X Mass Unfollow Tool Build
12,222 tokens

Model 9:04 AM
ThinkingThoughts
Expand to view model thoughts

chevron_right
The reason it likely didn't work is that X (Twitter) uses "Virtual Scrolling." This means the website only loads about 10–15 people on the screen at a time. If you tell the script to unfollow 50, but only 10 are visible, it gets stuck.
Also, X frequently changes their internal button IDs.
Here is an updated, more robust version of the content.js. It includes Auto-Scrolling (to find more people) and better error checking.
Updated content.js
Replace your old content.js with this code. I have added comments to explain how it handles X's dynamic layout.
code
JavaScript
chrome.runtime.onMessage.addListener((request, sender, sendResponse) => {
  if (request.action === "START_UNFOLLOW") {
    console.log("Starting batch: " + request.limit);
    unfollowBatch(request.limit);
  }
});

// Helper function to wait (delay)
const wait = (ms) => new Promise(r => setTimeout(r, ms));

async function unfollowBatch(limit) {
  let count = 0;
  let retryCount = 0;

  while (count < limit) {
    // 1. Find all "Following" buttons on the screen
    // We look for the button that says "Following"
    let buttons = Array.from(document.querySelectorAll('[role="button"]'))
      .filter(btn => btn.innerText === "Following" || btn.getAttribute('data-testid')?.includes('unfollow'));

    if (buttons.length === 0) {
      // If no buttons found, scroll down to load more
      console.log("No more buttons found, scrolling...");
      window.scrollBy(0, 800);
      await wait(2000); // Wait for X to load new users
      retryCount++;
      
      if (retryCount > 5) {
        alert("Reached the end of the list or X is blocking the script.");
        break;
      }
      continue;
    }

    for (const btn of buttons) {
      if (count >= limit) break;

      try {
        // Find the parent cell to check if they follow you
        const userCell = btn.closest('[data-testid="UserCell"]');
        const followsYou = userCell ? userCell.innerText.includes("Follows you") : false;

        // LOGIC: Only unfollow if they DON'T follow you
        if (!followsYou) {
          btn.scrollIntoView({ block: 'center' });
          btn.click(); // Click the "Following" button
          
          await wait(1000); // Wait for the confirmation popup

          // 2. Find the "Unfollow" confirmation button in the popup
          const confirmBtn = document.querySelector('[data-testid="confirmationSheetConfirm"]');
          
          if (confirmBtn) {
            confirmBtn.click();
            count++;
            console.log(`✅ Unfollowed: ${count}/${limit}`);
            
            // 3. Human-like delay (IMPORTANT: To prevent account ban)
            const delay = Math.floor(Math.random() * 3000) + 4000; // 4-7 seconds
            await wait(delay);
          }
        } else {
          console.log("Skipping follower...");
        }
      } catch (err) {
        console.error("Error clicking button:", err);
      }
    }
    
    // Scroll down after processing the current batch of visible buttons
    window.scrollBy(0, 1000);
    await wait(2000);
  }

  alert(`Done! Successfully unfollowed ${count} accounts.`);
}
Why this version works better:
btn.closest('[data-testid="UserCell"]'): It accurately finds the container for each user to check the "Follows you" text.
window.scrollBy: It automatically scrolls the page down. Because X only shows a few people at a time, the script now "hunts" for more users as it goes.
Confirmation Logic: It waits specifically for the confirmationSheetConfirm (the red "Unfollow" button that pops up).
Better Selectors: Instead of relying on just one ID, it looks for any button containing the text "Following".
How to test if it's working (Debugging):
If you click the button and nothing happens, follow these steps to see the "error":
Open your Following page on X.
Press F12 (or Right-click > Inspect) to open the Developer Tools.
Click the Console tab at the top.
Now, click your Extension button and choose "Unfollow 10".
Look at the console. It will print things like:
Starting batch: 10
✅ Unfollowed: 1/10
No more buttons found, scrolling...
Critical Reminder on "Detection":
If you try to unfollow 100 people in 1 minute, X will lock your account for "suspicious automation."
This script includes a 4–7 second delay per person.
Unfollowing 100 people will take about 10 minutes.
Do not close the tab while it is running.
Does it detect "Followers"?
Yes, in this line:
const followsYou = userCell.innerText.includes("Follows you");
If the script sees the small "Follows you" badge next to their name, it skips them so you don't accidentally unfollow your friends.
Model 9:12 AM
ThinkingThoughts
Expand to view model thoughts

chevron_right
To add a custom input field so you can specify exactly how many accounts to unfollow, you need to update the popup.html and popup.js files.
This version also includes Auto-Scrolling logic, because X only loads a few accounts at a time. Without scrolling, the script would stop after unfollowing just the first few visible people.
1. Updated popup.html
Add a numeric input field and a "Start" button.
code
Html
<!DOCTYPE html>
<html>
<head>
  <style>
    body { width: 220px; padding: 15px; font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; }
    h3 { margin-top: 0; font-size: 16px; color: #1da1f2; }
    .input-group { margin-bottom: 10px; }
    label { font-size: 12px; display: block; margin-bottom: 4px; }
    input { width: 100%; padding: 8px; box-sizing: border-box; border: 1px solid #ccc; border-radius: 4px; }
    button { width: 100%; margin-top: 10px; padding: 10px; background: #1da1f2; color: white; border: none; border-radius: 20px; cursor: pointer; font-weight: bold; }
    button:hover { background: #1a91da; }
    .status { margin-top: 10px; font-size: 11px; color: #666; text-align: center; }
  </style>
</head>
<body>
  <h3>X Mass Unfollower</h3>
  <div class="input-group">
    <label for="customAmount">Amount to unfollow:</label>
    <input type="number" id="customAmount" value="10" min="1" max="400">
  </div>
  <button id="startBtn">Start Unfollowing</button>
  <div class="status" id="status">Navigate to your "Following" list to start.</div>
  <script src="popup.js"></script>
</body>
</html>
2. Updated popup.js
This script captures the custom number and sends it to the page.
code
JavaScript
document.getElementById('startBtn').addEventListener('click', async () => {
  const amount = parseInt(document.getElementById('customAmount').value);
  const status = document.getElementById('status');
  
  if (isNaN(amount) || amount <= 0) {
    alert("Please enter a valid number.");
    return;
  }

  const [tab] = await chrome.tabs.query({ active: true, currentWindow: true });
  
  // Check if user is on the right page
  if (tab.url.includes('/following')) {
    chrome.tabs.sendMessage(tab.id, { action: "START_UNFOLLOW", limit: amount });
    status.innerText = `Unfollowing ${amount} accounts...`;
    status.style.color = "orange";
  } else {
    alert("Error: You must be on your 'Following' page (x.com/yourname/following).");
  }
});
[2][3]
3. Updated content.js
This script handles the automation, detects if they follow you back, and scrolls automatically to find more users.
code
JavaScript
chrome.runtime.onMessage.addListener((request, sender, sendResponse) => {
  if (request.action === "START_UNFOLLOW") {
    unfollowProcess(request.limit);
  }
});

async function unfollowProcess(limit) {
  let count = 0;
  console.log(`[Tool] Starting to unfollow ${limit} users...`);

  while (count < limit) {
    // 1. Find all visible user cells
    const userCells = Array.from(document.querySelectorAll('[data-testid="UserCell"]'));
    
    if (userCells.length === 0) {
      console.log("[Tool] No users found. Scrolling to load more...");
      window.scrollBy(0, 1000);
      await new Promise(r => setTimeout(r, 2000));
      continue; 
    }

    for (const cell of userCells) {
      if (count >= limit) break;

      // DETECT: Does this person follow you? 
      // X adds a "Follows you" text badge for mutuals.
      const followsYou = cell.innerText.includes("Follows you");

      if (!followsYou) {
        // Find the button that handles unfollowing
        const unfollowBtn = cell.querySelector('[data-testid$="-unfollow"]') || 
                           Array.from(cell.querySelectorAll('[role="button"]')).find(b => b.innerText === "Following");

        if (unfollowBtn) {
          unfollowBtn.scrollIntoView({ block: 'center' });
          unfollowBtn.click();
          
          // Wait for confirmation modal
          await new Promise(r => setTimeout(r, 800));
          const confirmBtn = document.querySelector('[data-testid="confirmationSheetConfirm"]');
          
          if (confirmBtn) {
            confirmBtn.click();
            count++;
            console.log(`[Tool] Progress: ${count}/${limit}`);
            
            // SAFETY DELAY: Randomized to avoid bot detection (5-10 seconds)
            const waitTime = Math.floor(Math.random() * 5000) + 5000;
            await new Promise(r => setTimeout(r, waitTime));
          }
        }
      }
    }

    // Scroll down to fetch the next batch of users from X's servers
    window.scrollBy(0, window.innerHeight);
    await new Promise(r => setTimeout(r, 2500));
    
    // Safety check to prevent infinite loops if we reach the bottom
    if ((window.innerHeight + window.scrollY) >= document.body.offsetHeight) {
      console.log("[Tool] Reached the end of the list.");
      break;
    }
  }

  alert(`Process Complete! Unfollowed ${count} users.`);
}
⚠️ Important Usage Tips:
Tab Focus: Keep the X tab open and visible while the script is running. If you switch tabs, Chrome may "pause" the script to save memory.
Safety First: X monitors "unfollow-churn."
Max per day: Stay under 100–150 for account safety.[4]
The Delay: The code uses a 5–10 second delay per action. This is intentional. Reducing this delay will make the process faster but significantly increases the risk of your account being suspended.
Language: This script checks for the English text "Follows you". If your X interface is in another language (e.g., Spanish "Te sigue"), you must change that string in the content.js file.
Sources
help
medium.com
medium.com
followeraudit.com
blackhatworld.com
Google Search Suggestions
Display of Search Suggestions is required when using Grounding with Google Search. Learn more
Google logo
Model 9:15 AM
ThinkingThoughts
Expand to view model thoughts

chevron_right
To add the watermark, we will update the popup.html to include a footer link and popup.js to ensure the link opens correctly in a new tab.
I have also styled it to match the X (Twitter) aesthetic.
1. Updated popup.html
Added a footer section with your credit and a link.
code
Html
<!DOCTYPE html>
<html>
<head>
  <style>
    body { width: 220px; padding: 15px; font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif; background-color: #fff; }
    h3 { margin-top: 0; font-size: 16px; color: #1da1f2; margin-bottom: 15px; }
    .input-group { margin-bottom: 15px; }
    label { font-size: 12px; display: block; margin-bottom: 5px; color: #555; }
    input { width: 100%; padding: 10px; box-sizing: border-box; border: 1px solid #e1e8ed; border-radius: 6px; outline: none; }
    input:focus { border-color: #1da1f2; }
    button { width: 100%; padding: 12px; background: #1da1f2; color: white; border: none; border-radius: 25px; cursor: pointer; font-weight: bold; font-size: 14px; transition: background 0.2s; }
    button:hover { background: #1a91da; }
    .status { margin-top: 12px; font-size: 11px; color: #657786; text-align: center; min-height: 15px; }
    
    /* Watermark Styling */
    .watermark {
      margin-top: 20px;
      padding-top: 10px;
      border-top: 1px solid #f0f3f4;
      text-align: center;
      font-size: 11px;
    }
    .watermark a {
      color: #1da1f2;
      text-decoration: none;
      font-weight: bold;
    }
    .watermark a:hover {
      text-decoration: underline;
    }
  </style>
</head>
<body>
  <h3>X Mass Unfollower</h3>
  
  <div class="input-group">
    <label for="customAmount">Amount to unfollow:</label>
    <input type="number" id="customAmount" value="10" min="1" max="400">
  </div>

  <button id="startBtn">Start Unfollowing</button>
  
  <div class="status" id="status">Ready</div>

  <div class="watermark">
    Built by <a href="https://x.com/0xDamisi" id="authorLink">@0xDamisi</a>
  </div>

  <script src="popup.js"></script>
</body>
</html>
2. Updated popup.js
In Chrome extensions, links in popups sometimes don't open automatically. We add a listener to handle the redirect to your profile.
code
JavaScript
document.getElementById('startBtn').addEventListener('click', async () => {
  const amount = parseInt(document.getElementById('customAmount').value);
  const status = document.getElementById('status');
  
  if (isNaN(amount) || amount <= 0) {
    alert("Please enter a valid number.");
    return;
  }

  const [tab] = await chrome.tabs.query({ active: true, currentWindow: true });
  
  if (tab.url.includes('/following')) {
    chrome.tabs.sendMessage(tab.id, { action: "START_UNFOLLOW", limit: amount });
    status.innerText = `Unfollowing ${amount} users...`;
    status.style.color = "#f45d22"; // Orange for "In Progress"
  } else {
    alert("Please navigate to your 'Following' page first.");
  }
});

// Logic to make the watermark link work
document.getElementById('authorLink').addEventListener('click', (e) => {
  e.preventDefault();
  chrome.tabs.create({ url: "https://x.com/0xDamisi" });
});
3. Updated content.js (Final Version)
Make sure this version is in your file to ensure it detects your followers correctly and scrolls properly.
code
JavaScript
chrome.runtime.onMessage.addListener((request, sender, sendResponse) => {
  if (request.action === "START_UNFOLLOW") {
    unfollowLogic(request.limit);
  }
});

async function unfollowLogic(limit) {
  let count = 0;
  console.log(`%c [0xDamisi Tool] Initiating batch: ${limit}`, "color: #1da1f2; font-weight: bold;");

  while (count < limit) {
    // 1. Get all user rows visible on screen
    const userCells = Array.from(document.querySelectorAll('[data-testid="UserCell"]'));
    
    if (userCells.length === 0) {
      window.scrollBy(0, 1000);
      await new Promise(r => setTimeout(r, 2000));
      continue;
    }

    for (const cell of userCells) {
      if (count >= limit) break;

      // 2. DETECT: Skip if they follow you back
      const followsYou = cell.innerText.includes("Follows you");

      if (!followsYou) {
        // Find the "Following" button
        const btn = cell.querySelector('[data-testid$="-unfollow"]') || 
                    Array.from(cell.querySelectorAll('[role="button"]')).find(b => b.innerText === "Following");

        if (btn) {
          btn.scrollIntoView({ block: 'center' });
          btn.click();
          
          await new Promise(r => setTimeout(r, 600)); // Wait for modal
          
          const confirm = document.querySelector('[data-testid="confirmationSheetConfirm"]');
          if (confirm) {
            confirm.click();
            count++;
            console.log(`[${count}/${limit}] Unfollowed account.`);
            
            // Randomized human-like delay (5 to 9 seconds)
            const delay = Math.floor(Math.random() * 4000) + 5000;
            await new Promise(r => setTimeout(r, delay));
          }
        }
      }
    }

    // 3. Scroll to load more content
    window.scrollBy(0, window.innerHeight);
    await new Promise(r => setTimeout(r, 2500));

    // Stop if we hit the bottom of the page
    if ((window.innerHeight + window.scrollY) >= document.body.offsetHeight) {
      console.log("[0xDamisi Tool] Reached the end of list.");
      break;
    }
  }

  alert(`Done! Successfully unfollowed ${count} accounts who don't follow you back.`);
}
Summary of Changes:
Watermark: A nice "Built by @0xDamisi" footer is now at the bottom of your extension popup.
Safety: The script still checks for the "Follows you" text so it doesn't unfollow people who follow you back.
Interactivity: Clicking @0xDamisi in the extension will open your Twitter profile in a new browser tab.
Visuals: The popup now looks more like an official Twitter tool with the #1da1f2 blue color scheme.
Model 9:28 AM
ThinkingThoughts
Expand to view model thoughts

chevron_right
To build a Cross-Checker, we need to rely on X’s built-in "Follows you" badge. When you are on your Following page, X explicitly labels accounts that follow you back with a small grey tag that says "Follows you".
This script will scan your following list, identify anyone without that tag (meaning they don't follow you back), and unfollow them up to your specified limit.
1. Updated popup.html
We’ll refine the UI to make it clear this is a "Not-Following-Back" checker.
code
Html
<!DOCTYPE html>
<html>
<head>
  <style>
    body { width: 250px; padding: 15px; font-family: -apple-system, sans-serif; background-color: #f7f9f9; }
    h3 { margin: 0 0 10px 0; font-size: 16px; color: #1da1f2; text-align: center; }
    p { font-size: 11px; color: #657786; margin-bottom: 15px; text-align: center; }
    .input-box { margin-bottom: 15px; }
    input { width: 100%; padding: 10px; border: 1px solid #ccd6dd; border-radius: 8px; box-sizing: border-box; }
    button { width: 100%; padding: 12px; background: #000; color: #fff; border: none; border-radius: 25px; cursor: pointer; font-weight: bold; }
    button:hover { background: #333; }
    .status { margin-top: 10px; font-size: 12px; text-align: center; color: #1da1f2; font-weight: bold; }
    .watermark { margin-top: 15px; padding-top: 10px; border-top: 1px solid #e1e8ed; text-align: center; font-size: 11px; }
    .watermark a { color: #1da1f2; text-decoration: none; font-weight: bold; }
  </style>
</head>
<body>
  <h3>Cross-Checker</h3>
  <p>Removes users who don't follow you back.</p>
  
  <div class="input-box">
    <input type="number" id="unfollowLimit" value="10" min="1">
  </div>

  <button id="runCheck">Start Cross-Check</button>
  
  <div class="status" id="displayStatus">Ready</div>

  <div class="watermark">
    Built by <a href="#" id="devLink">@0xDamisi</a>
  </div>

  <script src="popup.js"></script>
</body>
</html>
2. Updated popup.js
Handles the logic of starting the script and opening your link.
code
JavaScript
document.getElementById('runCheck').addEventListener('click', async () => {
  const limit = parseInt(document.getElementById('unfollowLimit').value);
  const [tab] = await chrome.tabs.query({ active: true, currentWindow: true });

  if (tab.url.includes('/following')) {
    document.getElementById('displayStatus').innerText = "Running... check Console (F12)";
    chrome.tabs.sendMessage(tab.id, { action: "CROSS_CHECK_UNFOLLOW", limit: limit });
  } else {
    alert("Error: Go to your FOLLOWING page first!");
  }
});

document.getElementById('devLink').addEventListener('click', () => {
  chrome.tabs.create({ url: "https://x.com/0xDamisi" });
});
3. Updated content.js (The Engine)
This version uses a "Cross-Check" logic: it looks for the UserCell, checks for the "Follows you" text, and only triggers the unfollow if that text is missing.
code
JavaScript
chrome.runtime.onMessage.addListener((request, sender, sendResponse) => {
  if (request.action === "CROSS_CHECK_UNFOLLOW") {
    crossCheck(request.limit);
  }
});

async function crossCheck(limit) {
  let unfollowedCount = 0;
  console.log(`%c [0xDamisi Cross-Checker] Target: ${limit} non-followers`, "color: #1da1f2; font-weight: bold;");

  while (unfollowedCount < limit) {
    // 1. Grab all user cards currently visible
    const users = Array.from(document.querySelectorAll('[data-testid="UserCell"]'));
    
    if (users.length === 0) {
    
