<template>
  <div>
    <select id="devices" v-model="selectedDeviceId">
      <option v-for="device in devices" :key="device.deviceId" :value="device.deviceId">
        {{ device.label || `カメラデバイス ${device.deviceId}` }}
      </option>
    </select>
    <button @click="resetScanner" style="margin-left: 2px;">適用</button>

    <div class="checkbox-container">
      <div 
				v-for="(result, index) in results" 
        :key="index" 
        class="square" 
        :class="{ checked: result !== ''}" 
        :style="index === 2 ? { marginRight: '1rem' } : {}"
      >
      </div>
    </div>
    <div class="camera-container">
      <video ref="video" autoplay></video>
      <canvas ref="canvas" class="overlay-canvas"></canvas>
    </div>

    <h3>読み取った値</h3>
    <ul>
      <li v-for="(result, index) in results" :key="index">
        {{ result }}
      </li>
    </ul>
    <h3>型式</h3>
		<p>{{ carType }}</p>


  </div>
</template>

<script>
import { BrowserMultiFormatReader, NotFoundException } from '@zxing/library';
import { ref, onMounted, onBeforeUnmount, watch } from 'vue';

const isStructured = (rawBytes) => {
  return (rawBytes[0] & 0xf0) === 0x30
}
const getNumber = (rawBytes) => {
  return rawBytes[0] & 0x0f
}
const getTotal = (rawBytes) => {
  return (rawBytes[1] >> 4) + 1
}

const is4thQr = (text) => {
  return text.split("/").length == 2 && text.split("/")[0] == "2";
};

const is5thQr = (text) => {
	//全角チェックの正規表現
	const zenkakuNumber = /^[\uFF10-\uFF19]$/;
	return text.split("/").length == 5 && zenkakuNumber.test(text.split("/")[0]);
};

export default {
  name: 'MultiQrCodeReader',
  setup() {

    const codeReader = new BrowserMultiFormatReader();
    const video = ref(null);
    const canvas = ref(null);
    const selectedDeviceId = ref(null);
    const devices = ref([]);
    const results = ref(["","","","",""]); // 読み取った結果を保存する配列
    const scanning = ref(false);
    const carType = ref("");


    // デバイスリストの取得
    const getDevices = async () => {
      const deviceInfos = await navigator.mediaDevices.enumerateDevices();
      devices.value = deviceInfos.filter((device) => device.kind === 'videoinput');
      if (devices.value.length > 0) {
        selectedDeviceId.value = devices.value[0].deviceId; // 最初のデバイスを選択
      }
    };

    // QRコードの読み取り開始
    const startScanning = async () => {
      if (!selectedDeviceId.value || scanning.value) return;
      scanning.value = true;

      try {
        await codeReader.decodeFromVideoDevice(
          selectedDeviceId.value,
          video.value,
          (result, err) => {
            if (result) {
              drawGuide(result); // ガイドを描画
              const text = result.getText();
              if (!results.value.includes(text)) {
                const rawBytes = result.getRawBytes();
								if (!isStructured(rawBytes)) {
									console.log("分割QRコードではありません")
									return;
								}
								const num = getNumber(rawBytes)
								const total = getTotal(rawBytes)

								if (total == 3 && num == 0) {
									results.value[0] = text;
								} else if (total == 3 && num == 1) {
									results.value[1] = text;
								} else if (total == 3 && num == 2) {
									results.value[2] = text;
								} else if (total == 2 && num == 0 && is4thQr(text)) {
									results.value[3] = text;
								} else if (total == 2 && num == 1 && is5thQr(text)) {
									results.value[4] = text;
								} else {
                  console.error('想定しないQRコードが検出されました');
                  return
                }

                if (results.value[0] && results.value[1] && results.value[2] && results.value[3] && results.value[4]) {
								  console.log('QRコード読み取り完了');

								  scanning.value = false;

								  const qr2 = results.value[3] + results.value[4];
								  const qr3 = results.value[0] + results.value[1] + results.value[2];
								  carType.value = qr3.split("/")[5];
							  }
              }
            } else if (err && !(err instanceof NotFoundException)) {
              console.error('QRコード読み取りエラー:', err.message);
            }
          }
        );
      } catch (err) {
        console.error('カメラ初期化エラー:', err);
      }
    };

    const drawGuide = (result) => {
      const canvasElement = canvas.value;
      const context = canvasElement.getContext('2d');
      canvasElement.width = video.value.videoWidth;
      canvasElement.height = video.value.videoHeight;

      setTimeout(() => {
        context.clearRect(0, 0, canvasElement.width, canvasElement.height);
      }, 1000);

      const points = result.resultPoints; // QRコードの頂点情報を取得

      context.beginPath();
      context.moveTo(points[0].x, points[0].y);

			const height = (points[1].y - points[0].y) * 1.3;
			const width = (points[2].x - points[1].x) * 1.3;
      const pointX = points[0].x - (points[2].x - points[1].x) * 0.1
      const pointY = points[0].y - (points[1].y - points[0].y) * 0.3
			context.rect(pointX, pointY, width, height);

      context.closePath();
      context.lineWidth = 10;
      context.strokeStyle = 'red'; // ガイドの色
      context.stroke()
    };

    // リセット機能
    const resetScanner = async () => {
      // ビデオストリームの停止
      let stream = video.value?.srcObject;
        if (stream) {
          const tracks = stream.getTracks();
          tracks.forEach(track => track.stop());
      }

      // 結果のリセット
      results.value = ["", "", "", "", ""];
      carType.value = "";
      scanning.value = false;
      
      await navigator.mediaDevices.getUserMedia({
        video: {
          deviceId: selectedDeviceId.value,
          width: { ideal: 3840 }, // 理想的な幅
          height: { ideal: 2080 }, // 理想的な高さ
          facingMode: 'environment', // 背面カメラを使用（モバイル用）
          advanced: [
            { focusMode: "continuous" } // 自動フォーカスをリクエスト
          ],
        },
      });
      startScanning();
      console.log('スキャナーをリセットしました');
    };

    // 初期化処理
    onMounted(async () => {
      await getDevices();

      if (selectedDeviceId.value) {
        // カメラの解像度設定
        await navigator.mediaDevices
          .getUserMedia({
            video: {
              deviceId: selectedDeviceId.value,
              width: { ideal: 3840 }, // 理想的な幅
              height: { ideal: 2080 }, // 理想的な高さ
              facingMode: 'environment', // 背面カメラを使用（モバイル用）
              advanced: [
                { focusMode: "continuous" } // 自動フォーカスをリクエスト
              ],
            },
          })
        startScanning();
      } 
    });

    // クリーンアップ処理
    onBeforeUnmount(() => {
      codeReader.reset();
    });

    return {
      video,
      canvas,
      devices,
      selectedDeviceId,
      resetScanner,
      results,
      carType,
    };
  },
};
</script>

<style scoped>
video {
  width: 100%;
  max-width: 640px;
  height: auto;
  border: 1px solid #ccc;
  margin-top: 10px;
}

canvas {
  width: 100%;
  max-width: 640px;
  height: auto;
}

h3 {
  margin-top: 20px;
}

ul {
  list-style-type: none;
  padding: 0;
}

li {
  margin: 5px 0;
  padding: 10px;
  border: 1px solid #ddd;
  border-radius: 4px;
}

.checkbox-container {
  display: flex;
  gap: 10px;
  flex-direction: row;
}

.square {
  width: 3rem;
  height: 3rem;
  border: 1px solid #000;
  display: flex;
  justify-content: center;
  align-items: center;
  cursor: pointer;
  transition: background-color 0.3s;
}

.square.checked {
  background-color: #4caf50; /* チェックがついた場合の色 */
}

.camera-container {
  position: relative;
}

.overlay-canvas {
  position: absolute;
  top: 0;
  left: 0;
  pointer-events: none;
}

</style>
