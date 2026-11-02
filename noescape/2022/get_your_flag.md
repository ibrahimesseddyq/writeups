# Get_your_flag

**CTF:** NOESCAPE 2022  
**Category:** web  
**Original writeup:** <https://blog.aravindha.in/noescape-live/> — by **blog.aravindha.in**. All credit goes to the original author; this is an archived copy.

---

Get your flag

This was a kinda code review challenge with a parameter pollution exploit. https://crack-admin.herokuapp.com/ index.js

const express = require("express"); const bodyParser = require("body-parser"); const { inviteCode, flag } = require("./secret"); const app = express(); app.use(bodyParser.text()); const port = process.env.PORT || 3000; const baseUser = { picture: "default.png" }; function createAdmin() { return `Creating admin account, flag is ${flag}`; } function createUser() { return "Creating user account"; } app.post("/", (req, res) => { let user; try { user = JSON.parse(req.body); } catch (e) { return res.status(400).json({ message: "Invalid Request body" }); } if (user.isAdmin && user.inviteCode !== inviteCode) { res.send("No invite code? No admin!"); } else { let newUser = Object.assign(baseUser, user); if (newUser.isAdmin) { res.send(createAdmin(newUser)); } else { res.send(createUser(newUser)); } } }); app.listen(port, () => { console.log(`App listening on port ${port}`); });

This Object.Assign is vulnerable to parameter pollution and you can manipulate the value through injecting payload like {" proto ":{"isAdmin":true}} which will give you the flag.
