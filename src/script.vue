<!-- Use preprocessors via the lang attribute! e.g. <template lang="pug"> -->
<template>
  <div id="app">
    <div class="bar-bg" v-show="entries.length"></div>
    <div class="bar" ref="bar" v-show="entries.length"></div>
    <div class="entries" v-if="entries.length">
      <div
        class="entry"
        v-for="(entry, index) in entries"
        tabindex="0"
        @keydown.enter="copy(entry.otp)"
        @keydown.prevent.space="copy(entry.otp)"
        @keydown.delete="deleteEntry(index)"
        @keydown.ctrl.c="isCollapsed() && copy(entry.otp)"
        @keydown.stop.up="focusPrev"
        @keydown.stop.left="focusPrev"
        @keydown.stop.down="focusNext"
        @keydown.stop.right="focusNext"
      >
        <div class="entry-left">
          <div v-if="entry.label">{{ entry.label }}</div>
          <div>secret: {{ entry.secret }}</div>
          <div v-if="entry.issuer">issuer: {{ entry.issuer }}</div>
          <div>{{ new Date(entry.created_at) }}</div>
        </div>
        <div @click="copy(entry.otp)">
          <h1>{{ entry.otp }}</h1>
        </div>
      </div>
      <h1 title="Delete a entry by DELETE key">
        Copy a OTP by
        <span class="key">Enter</span>
        ,
        <span class="key">Space</span>
        ,
        <span class="nowrap">
          <span class="key">Ctrl</span>
          +
          <span class="key">C</span>
        </span>
        , click&nbsp;a&nbsp;OTP
      </h1>
      <h1>
        Delete a card by
        <span class="key">Delete</span>
      </h1>
      <h1>
        Select a card by
        <span class="key">←</span>
        <span class="key">↑</span>
        <span class="key">→</span>
        <span class="key">↓</span>
      </h1>
    </div>
    <div
      v-else
      class="initial"
      title="If you paste QR image, add some white padding around the QR image by Win + Shift + S."
    >
      <h1>
        Paste a secret key, URL, QR image by
        <span class="nowrap">
          <span class="key">Ctrl</span>
          +
          <span class="key">V</span>
        </span>
      </h1>
    </div>
    <a
      href="https://github.com/lab8-tomofi/online-totp-generator"
      class="github-corner"
      aria-label="View source on GitHub"
      target="_blank"
    >
      <svg
        width="80"
        height="80"
        viewBox="0 0 250 250"
        :style="{
          fill: '#64ceaa',
          color: '#fff',
          position: 'fixed',
          top: entries.length ? '8px' : 0,
          border: 0,
          right: 0
        }"
        aria-hidden="true"
      >
        <path d="M0,0 L115,115 L130,115 L142,142 L250,250 L250,0 Z"></path>
        <path
          d="M128.3,109.0 C113.8,99.7 119.0,89.6 119.0,89.6 C122.0,82.7 120.5,78.6 120.5,78.6 C119.2,72.0 123.4,76.3 123.4,76.3 C127.3,80.9 125.5,87.3 125.5,87.3 C122.9,97.6 130.6,101.9 134.4,103.2"
          fill="currentColor"
          style="transform-origin: 130px 106px"
          class="octo-arm"
        ></path>
        <path
          d="M115.0,115.0 C114.9,115.1 118.7,116.5 119.8,115.4 L133.7,101.6 C136.9,99.2 139.9,98.4 142.2,98.6 C133.8,88.0 127.5,74.4 143.8,58.0 C148.5,53.4 154.0,51.2 159.7,51.0 C160.3,49.4 163.2,43.6 171.4,40.1 C171.4,40.1 176.1,42.5 178.8,56.2 C183.1,58.6 187.2,61.8 190.9,65.4 C194.5,69.0 197.7,73.2 200.1,77.6 C213.8,80.2 216.3,84.9 216.3,84.9 C212.7,93.1 206.9,96.0 205.4,96.6 C205.1,102.4 203.0,107.8 198.3,112.5 C181.9,128.9 168.3,122.5 157.7,114.1 C157.9,116.9 156.7,120.9 152.7,124.9 L141.0,136.5 C139.8,137.7 141.6,141.9 141.8,141.8 Z"
          fill="currentColor"
          class="octo-body"
        ></path>
      </svg>
    </a>
  </div>
</template>

<script>
import jsQR from "https://cdn.skypack.dev/jsqr@1.4.0";

async function scanQRFromFile(file) {
  const img = new Image();
  img.src = URL.createObjectURL(file);
  await new Promise((resolve) => img.addEventListener("load", resolve));
  const canvas = new OffscreenCanvas(img.naturalWidth, img.naturalHeight);
  const ctx = canvas.getContext("2d");
  ctx.imageSmoothingEnabled = false;
  ctx.drawImage(img, 0, 0);
  const imageData = ctx.getImageData(0, 0, img.naturalWidth, img.naturalHeight);
  return jsQR(imageData.data, img.naturalWidth, img.naturalHeight)?.data;
}

async function scanQRFromFileWithPaddingAndWhiteBackground(file, padding = 32) {
  const img = new Image();
  img.src = URL.createObjectURL(file);
  await new Promise((resolve) => img.addEventListener("load", resolve));
  const width = img.naturalWidth + padding * 2;
  const height = img.naturalHeight + padding * 2;
  const canvas = new OffscreenCanvas(width, height);
  const ctx = canvas.getContext("2d");
  ctx.imageSmoothingEnabled = false;
  ctx.fillStyle = "white";
  ctx.fillRect(0, 0, width, height);
  ctx.drawImage(img, padding, padding);
  const imageData = ctx.getImageData(0, 0, width, height);
  return jsQR(imageData.data, width, height)?.data;
}

export default {
  data() {
    return {
      entries: JSON.parse(localStorage.getItem("entries")) ?? []
    };
  },
  mounted() {
    this.update();
    addEventListener("keydown", (e) => {
      if (["ArrowDown", "ArrowRight"].includes(e.key)) {
        if (!document.querySelector(".entry:first-of-type")) return;
        document.querySelector(".entry:first-of-type").focus();
      }
      if (["ArrowUp", "ArrowLeft"].includes(e.key)) {
        if (!document.querySelector(".entry:last-of-type")) return;
        document.querySelector(".entry:last-of-type").focus();
      }
    });
    addEventListener("paste", async (e) => {
      const text = e.clipboardData.getData("text");
      this.addText(text);
      if (!e.clipboardData.files.length) return;
      const file = e.clipboardData.files[0];
      let data = await scanQRFromFile(file);
      if (!data) data = await scanQRFromFileWithPaddingAndWhiteBackground(file);
      if (!data) {
        Toastify({
          text: "Could not scan the QR code.",
          duration: 10000,
          style: {
            background: "linear-gradient(to right, #ff5f6d, #ffc371)"
          },
          gravity: "bottom"
        }).showToast();
        return;
      }
      this.addText(data);
    });
    this.$nextTick().then(() => {
      if (!document.querySelector(".entry:first-of-type")) return;
      document.querySelector(".entry:first-of-type").focus();
    });
  },
  methods: {
    update() {
      const ms = 1000 * 30;
      const duration = ms - (Date.now() % ms);
      setTimeout(this.update, duration);
      this.$refs.bar.animate(
        [
          { transform: `translateX(-${((ms - duration) / ms) * 100}%)` },
          { transform: `translateX(-100%)` }
        ],
        {
          duration
        }
      );
      this.updateOtp();
    },
    updateOtp() {
      for (const entry of this.entries) {
        const totp = new OTPAuth.TOTP(entry);
        entry.otp = totp.generate();
      }
    },
    addEntry(entry) {
      entry.created_at = new Date();
      const totp = new OTPAuth.TOTP(entry);
      entry.otp = totp.generate();
      this.entries.unshift(entry);
      localStorage.setItem("entries", JSON.stringify(this.entries));
      this.copy(entry.otp);
      this.$nextTick().then(() => {
        document.querySelector(".entry:first-of-type").focus();
        scrollTo({ top: 0, behavior: "smooth" });
      });
    },
    deleteEntry(index) {
      this.$delete(this.entries, index);
      localStorage.setItem("entries", JSON.stringify(this.entries));
    },
    addText(text) {
      if (!text) return;
      text = text.trim();
      if (/^[2-7]{6}$/.test(text)) return;
      if (/^[A-Z2-7]+$/.test(text)) this.addEntry({ secret: text });
      try {
        const url = new URL(text);
        const entry = Object.fromEntries(url.searchParams);
        entry.label = decodeURIComponent(url.pathname.split("/").at(-1));
        this.addEntry(entry);
      } catch (e) {
        console.error(e);
      }
    },
    copy(text) {
      const activeElement = document.activeElement;
      Toastify({ text, gravity: "bottom" }).showToast();
      const input = document.body.appendChild(document.createElement("input"));
      input.value = text;
      input.select();
      document.execCommand("copy");
      input.parentNode.removeChild(input);
      activeElement.focus();
    },
    isCollapsed() {
      return getSelection().isCollapsed;
    },
    focusPrev(e) {
      if (!e.target.previousElementSibling) return;
      e.target.previousElementSibling.focus();
    },
    focusNext(e) {
      if (!e.target.nextElementSibling) return;
      e.target.nextElementSibling.focus();
    }
  }
};
</script>

<!-- Use preprocessors via the lang attribute! e.g. <style lang="scss"> -->
<style>
body {
  margin: 0;
}

#app {
  text-align: center;
  color: #2c3e50;
}

.bar-bg {
  position: sticky;
  top: 0;
  height: 8px;
  background: lightgreen;
}

.bar {
  position: sticky;
  top: 0;
  margin-top: -8px;
  height: 8px;
  background: green;
  will-change: transform;
}

.entries {
  margin: 8px;
}

.entry {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 16px;
  margin: 16px auto;
  border: 2px solid #fafafa;
  border-radius: 8px;
  padding: 8px;
  max-width: 768px;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.26);
  outline-color: green;
}

.entry:focus,
.entry:focus-visible {
  border: 2px solid green;
  outline: 2px solid green;
}

.entry-left {
  text-align: left;
}

.key {
  border: 1px solid #19191c33;
  border-radius: 4px;
  padding: 8px 16px;
  line-height: 2.2;
  background: #19191c0d;
}

.nowrap {
  white-space: nowrap;
}

.initial {
  display: flex;
  align-items: center;
  justify-content: center;
  height: 100vh;
}

.github-corner:hover .octo-arm {
  animation: octocat-wave 560ms ease-in-out;
}
@keyframes octocat-wave {
  0%,
  100% {
    transform: rotate(0);
  }
  20%,
  60% {
    transform: rotate(-25deg);
  }
  40%,
  80% {
    transform: rotate(10deg);
  }
}
@media (max-width: 500px) {
  .github-corner:hover .octo-arm {
    animation: none;
  }
  .github-corner .octo-arm {
    animation: octocat-wave 560ms ease-in-out;
  }
}
</style>
