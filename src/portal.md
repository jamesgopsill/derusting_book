# Submission Portal

[HTTP Service](./http.md) gave us a `GET /` that returns *something* — this chapter is that something: the actual page a user sees when they point a browser at a machine. It's deliberately tiny: a file picker, a button, and just enough JavaScript to PUT the chosen file straight to the machine that's serving the page. Stage gate: **visit two different machines' portals side by side and see the identical page served independently by each — no shared server, no dependency on any one machine being up.**

## The whole page

`assets/index.html` is the entire portal — no build step, no framework, just plain HTML/CSS/JS:

```html
<!DOCTYPE html>
<title>Upload</title>
<style>
  /* ...centered card, blue button, nothing fancy... */
</style>
<div class="c">
  <h1>Derusting</h1>
  <p><b>De</b>centralised T<b>rust</b>ed Manufactur<b>ing</b> in <b>Rust</b></p>
  <input type="file" id="f">
  <button onclick="u()">Upload G-code</button>
</div>
<script>
  async function u() {
    let file = f.files[0];
    if (!file) return alert('Select file');
    try {
      let r = await fetch('/', { method: 'PUT', body: file });
      alert(r.ok ? 'Sent' : 'Error');
    } catch (e) { alert('Fail'); }
  }
</script>
```

That's it — one file input, one button, one `fetch()` call. When you pick a file and click "Upload G-code", the browser PUTs the raw file bytes straight to `/` on whichever machine served the page.

## No server-side templating, no build step

Unlike most web apps, this page isn't rendered per-request — it's baked into the firmware binary at compile time. Back in [LWIP TCP](./lwip_tcp.md), `http.rs` pulls the file in with `include_str!` and wraps it in a fixed HTTP response, computed once, at compile time:

```rust
const HTML_STR: &str = include_str!("../assets/index.html");
pub const INDEX_HTML: &str = formatcp!(
    "HTTP/1.1 200 OK\r\n\
    Content-Type: text/html\r\n\
    Content-Length: {}\r\n\
    Connection: close\r\n\r\n\
    {}",
    HTML_STR.len(),
    HTML_STR
);
```

There's no templating, no per-request state, no session — `GET /` always returns the exact same bytes, computed once at build time and baked straight into the firmware image. That has a nice consequence: **every machine's portal is identical and entirely self-contained.** There's no shared "submission service" any of them depend on — each one serves its own copy of the same page from its own firmware, independently of every other machine on the network.

## No redirect-if-busy logic (yet)

Some decentralised systems would have a busy machine redirect a visitor to a less-loaded peer. Derusting doesn't do that today — `GET /` always serves the same static page regardless of load, and there's no logic anywhere in `handle_conn` that checks whether the machine is currently printing before serving the form. It's a reasonable next step (and one this book may cover in a future chapter), but don't go looking for it in the current source.

## Did it work?

With two machines flashed and reachable on the network, open both portals in separate tabs:

```
http://[MACHINE_A_IP]:8080
http://[MACHINE_B_IP]:8080
```

<p align="center">
  <img src="https://github.com/jamesgopsill/derusting_book/blob/main/src/assets/website.png?raw=true" height="400" alt="Website">
</p>

You should see byte-for-byte the same page from each — same title, same styling, same upload button — served entirely independently by two different pieces of hardware running two different copies of the same firmware. Try uploading a job through machine A's portal and machine B's portal separately: both accept it (you'll see each one's own `/ PUT request received` log line), completely unaware of what the other is doing at the HTTP layer. Everything that then keeps the two machines in sync — sharing the file, deciding who prints it — happens entirely over UDP, covered in [Gossiping](./gossip.md), not through anything to do with this page.
