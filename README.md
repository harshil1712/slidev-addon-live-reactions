
# Slidev Addon: Live Reactions

Enhance your Slidev presentations with real-time audience reactions!

## Features

- Add live reaction emojis to your Slidev presentations
- Real-time updates visible to all connected viewers
- Customizable emoji set
- Easy integration with existing Slidev projects

## Installation

```bash
npm install slidev-addon-live-reactions
```

## Usage

1. Add the following to your slides.md frontmatter:

```md
---
addons:
  - slidev-addon-live-reactions

liveReactions:
  server: ws://localhost:8787/ws
---
```

> [!NOTE]
> Make sure to replace `ws://localhost:8787/ws` with the actual WebSocket server URL. You can find the Server code in this [GitHub repository](https://github.com/harshil1712/slidev-realtime-feedback-do).


2. Add the component to `global.vue`
```vue
<template>
  <LiveReactions />
</template>
```

## Configuration

You can customize the available reactions by modifying your slides.md frontmatter:

```md
---
liveReactions:
  okay: 👀
  good: 👍
  great: ❤️
  mindBlown: 🤯
---
```

## Contributing

Contributions are welcome! Please feel free to open issues or submit pull requests.