<!DOCTYPE html>
<html>
  <head>
    <base target="_top">
    <style>
      :root { --navy:#14375e; --accent:#185FA5; --line:#d8dee6; --text:#1f2a37; --muted:#5b6b7d; }
      * { box-sizing: border-box; }
      body { font-family:-apple-system,"Segoe UI",Roboto,Arial,sans-serif; background:#f4f6f9;
        color:var(--text); margin:0; padding:24px 12px; line-height:1.55; }
      .card { max-width:640px; margin:0 auto; background:#fff; border:1px solid var(--line);
        border-radius:12px; overflow:hidden; box-shadow:0 2px 10px rgba(20,55,94,0.08); }
      .topbar { background:var(--navy); color:#fff; padding:18px 22px; }
      .topbar .dept { font-size:12px; letter-spacing:0.1em; text-transform:uppercase; opacity:0.8; }
      .topbar h1 { margin:2px 0 0; font-size:20px; font-weight:600; }
      .body { padding:22px 24px; }
      .body h2 { font-size:15px; color:var(--navy); margin:20px 0 6px; }
      .body p { margin:0 0 12px; }
      .updated { color:var(--muted); font-size:13px; margin-top:4px; }
      a { color:var(--accent); }
    </style>
  </head>
  <body>
    <div class="card">
      <div class="topbar">
        <div class="dept">Huntington Beach Police Department</div>
        <h1>Shift Bid — SMS Terms &amp; Conditions</h1>
      </div>
      <div class="body">
        <h2>Program</h2>
        <p>The Huntington Beach Police Department (HBPD) "Shift Bid" SMS program sends text messages
           to department sergeants for internal shift-bid scheduling only. These messages contain no
           marketing or promotional content.</p>

        <h2>Consent</h2>
        <p>By providing your mobile number to the department, you agree to receive shift-scheduling
           text messages from HBPD. Consent is not a condition of any purchase.</p>

        <h2>Message frequency</h2>
        <p>Message frequency is low — typically about one message per sergeant per bid cycle, with
           only a few bid cycles per year.</p>

        <h2>Cost</h2>
        <p>Message and data rates may apply, according to the terms of your mobile carrier plan.</p>

        <h2>Opting out and help</h2>
        <p>You can cancel at any time by replying <b>STOP</b> to any message; you will receive one
           confirmation message and no further texts. For help, reply <b>HELP</b> or contact us at
           the email below.</p>

        <h2>Carrier disclaimer</h2>
        <p>Carriers are not liable for delayed or undelivered messages.</p>

        <h2>Privacy</h2>
        <p>No mobile information is shared with third parties or affiliates for marketing or
           promotional purposes. Text messaging originator opt-in data and consent are not shared
           with any third parties. See our <a href="<?= pageUrl_('privacy') ?>">Privacy Policy</a>.</p>

        <h2>Contact</h2>
        <p>Questions can be directed to <a href="mailto:SWhite@hbpd.org">SWhite@hbpd.org</a>.</p>

        <p class="updated">Huntington Beach Police Department · Internal workforce scheduling use only.</p>
      </div>
    </div>
  </body>
</html>
