<template>
  <div
    class="container"
    v-if="png_info"
    style="
      width: 100%;
      max-height: 300px;
      overflow-y: scroll;
      position: relative;
    "
  >
    <table
      class="main-table"
      :style="{ borderBottomWidth: comment_info ? '0px' : '1px' }"
    >
      <tr v-for="(item, index) in png_info" style="position: relative">
        <td
          style="width: 65px; word-break: keep-all"
          v-if="item.keyword !== 'Comment'"
        >
          {{ item.keyword }}
        </td>
        <td style="width: max-content" v-if="item.keyword !== 'Comment'">
          {{ item.text }}
        </td>
      </tr>
    </table>
    <table
      v-if="comment_info"
      class="main-table"
      :style="{ borderTopWidth: comment_info ? '0px' : '1px' }"
    >
      <tr>
        <td colspan="3" style="text-align: center">Comment</td>
      </tr>
      <tr v-for="[key, value] in Object.entries(comment_info)">
        <td
          v-if="['prompt', 'uc'].includes(key)"
          style="width: 50px; text-align: center"
          colspan="1"
        >
          {{ key }}
        </td>
        <td v-else style="width: max-content" colspan="2">{{ key }}</td>

        <td
          v-if="['prompt', 'uc'].includes(key)"
          style="width: max-content"
          colspan="2"
        >
          {{ value }}
        </td>
        <td v-else style="width: 109px" colspan="1">{{ value }}</td>
      </tr>
    </table>
  </div>
  <div v-else>
    The PNG file does not contain relevant information. It might not have been
    generated directly by AI, or its metadata may have been erased.
  </div>
</template>

<style lang="scss">
html {
  font-size: 11px;
  font-family: sans-serif;
  border-radius: 6px;
  overflow: hidden;
}

// 滚动条尺寸
.container::-webkit-scrollbar {
  width: 5px;
  height: 10px;
}

body {
  padding: 0;
  margin: 0;
  color: transparent;

  // 亮色主题
  &[theme="LIGHT"],
  &[theme="LIGHTGRAY"] {
    color: black;

    // 列表左列
    .main-table > tr > td:nth-child(1) {
      background-color: rgb(227, 228, 229);
    }

    // 滚动条样式
    .container::-webkit-scrollbar-thumb {
      background: rgb(186, 186, 187);
      border-radius: 3px;

      &:hover {
        background: rgb(124, 124, 124);
        border-radius: 3px;
      }
    }

    table,
    table tr th,
    tr td {
      border: 1px solid white;
    }
  }

  // 暗色主题
  &[theme="GRAY"],
  &[theme="BLUE"],
  &[theme="PURPLE"],
  &[theme="DARK"],
  &[theme="undefined"] {
    color: white;

    // 列表左列
    .main-table > tr > td:nth-child(1) {
      background-color: rgb(43, 44, 47);
    }

    // 滚动条样式
    .container::-webkit-scrollbar-thumb {
      background: rgb(87, 88, 90);
      border-radius: 3px;

      &:hover {
        background: rgb(143, 144, 145);
        border-radius: 2.5px;
      }
    }

    table,
    table tr th,
    tr td {
      border: 1px solid black;
    }
  }
}

table {
  text-align: left;
  border-collapse: separate;
  border-spacing: 0;
}
td {
  padding: 2px 5px;
}
.main-table {
  width: 100%;
  word-break: break-all;
  word-wrap: break-word;
  line-height: 1.5rem;

  &:first-child {
    border-top-left-radius: 6px;
    border-top-right-radius: 6px;

    tr:first-child {
      border-top-left-radius: 6px;
      border-top-right-radius: 6px;

      > td:first-child {
        border-top-left-radius: 6px;
      }

      > td:last-child {
        border-top-right-radius: 6px;
      }
    }
  }

  &:last-child {
    border-bottom-left-radius: 6px;
    border-bottom-right-radius: 6px;

    tr:last-child {
      border-bottom-left-radius: 6px;
      border-bottom-right-radius: 6px;

      > td:first-child {
        border-bottom-left-radius: 6px;
      }

      > td:last-child {
        border-bottom-right-radius: 6px;
      }
    }
  }
}
</style>
<script setup>
const fs = require("fs");
import ExtractChunks from "png-chunks-extract";
import Text from "png-chunk-text";
import { ref } from "vue";
const extractMetadata = async (buf) => {
  let chunks = [];
  try {
    chunks = ExtractChunks(new Uint8Array(buf));
  } catch (err) {
    return chunks;
  }

  const textChunks = chunks
    .filter((chunk) => ["tEXt", "iTXt"].includes(chunk.name))
    .map((chunk) => {
      if (chunk.name === "iTXt") {
        let data = chunk.data.filter((c) => c != 0x0);
        const header = new TextDecoder().decode(data.slice(0, 11));
        if (header == "Description") {
          data = data.slice(11);
        }
        return {
          keyword: header == "Description" ? header : "Unknown",
          text: new TextDecoder().decode(data),
        };
      } else {
        return Text.decode(chunk.data);
      }
    });
  return textChunks;
};

const png_info = ref(null);
const comment_info = ref(null);
eagle.onPluginCreate(async (plugin) => {
  const item = await eagle.item.getSelected();
  if (item.length === 1) {
    const theme = await eagle.app.theme;
    document.body.setAttribute("theme", theme);
    console.log(theme);

    const item0 = item[0];

    fs.readFile(item0.filePath, async (err, buf) => {
      if (err) throw err;
      const info = await extractMetadata(buf.buffer);
      const comment = info.find((item) => item.keyword === "Comment");
      if (comment) {
        comment_info.value = JSON.parse(comment.text);
      }
      if (info.length != 0) {
        png_info.value = info;
      }
    });
  }
});

// Listen to theme changes
eagle.onThemeChanged((theme) => {
  document.body.setAttribute("theme", theme);
});
</script>
