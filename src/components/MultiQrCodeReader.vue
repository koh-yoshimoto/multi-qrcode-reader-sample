<template>
  <div>
    <h1>分割QRコードリーダー</h1>
    <video ref="video" autoplay muted playsinline></video>
    <p>読み取り中: {{ currentCount }}/5</p>
    <button @click="resetReader">リセット</button>
    <p v-if="finalResult">結合結果: {{ finalResult }}</p>
    <p v-if="error" class="error">{{ error }}</p>
  </div>
</template>

<script>
import { ref, reactive, onMounted, onBeforeUnmount } from 'vue';
import { BrowserMultiFormatReader } from '@zxing/library';

export default {
  name: 'SplitQrCodeReader',
  setup() {
    const video = ref(null);
    const error = ref(null);
    const currentCount = ref(0);
    const totalParts = 5; // QRコードの分割数
    const parts = reactive([]); // QRコードの各部分を格納
    const finalResult = ref(null);
    let codeReader = null;

    const startVideo = async () => {
      try {
        codeReader = new BrowserMultiFormatReader();
        const videoInputDevices = await codeReader.listVideoInputDevices();
        if (videoInputDevices.length === 0) {
          error.value = 'カメラが見つかりません';
          return;
        }

        const firstDeviceId = videoInputDevices[0].deviceId;

        codeReader.decodeFromVideoDevice(firstDeviceId, video.value, (result, err) => {
          if (result) {
            handleResult(result.getText());
          }
          if (err) {
            console.warn(err); // エラーは無視（読み取り中の失敗もあるため）
          }
        });
      } catch (err) {
        error.value = 'カメラを初期化できませんでした: ' + err.message;
      }
    };

    const stopVideo = () => {
      if (codeReader) {
        codeReader.reset();
      }
    };

    const handleResult = (text) => {
      if (!parts.includes(text)) {
        parts.push(text);
        currentCount.value = parts.length;

        // 全てのQRコードが揃ったら結合
        if (parts.length === totalParts) {
          stopVideo();
          finalResult.value = parts.join(''); // 全ての部分を結合
        }
      }
    };

    const resetReader = () => {
      parts.splice(0); // 部分データをクリア
      currentCount.value = 0;
      finalResult.value = null;
      error.value = null;
      startVideo(); // 再スタート
    };

    onMounted(() => {
      startVideo();
    });

    onBeforeUnmount(() => {
      stopVideo();
    });

    return {
      video,
      error,
      currentCount,
      finalResult,
      resetReader,
    };
  },
};
</script>

<style>
.error {
  color: red;
}
</style>
