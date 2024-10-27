<template>
  <div class="min-h-screen p-6 bg-gray-100">
    <h1 class="pt-12 mb-4 text-3xl font-bold text-blue-600">
      Card Prediction with Markov Chain
    </h1>

    <div class="flex items-center justify-center pb-6">
      <div
        class="w-full h-24 text-lg rounded-lg md:w-1/3"
        :class="prediction == 'red' ? 'bg-red-500' : 'bg-blue-500'"
        v-if="prediction"
      >
      </div>
    </div>

    <div class="flex flex-col items-center justify-center">
      <div class="flex w-full gap-2 md:w-2/3">
        <div
          class="w-1/2 h-32 bg-blue-500 rounded-lg shadow-md cursor-pointer drop-shadow-md"
          @click="submitCardResult('blue')"
        >
          Blue
        </div>
        <div
          class="w-1/2 h-32 bg-red-500 rounded-lg shadow-md cursor-pointer drop-shadow-md"
          @click="submitCardResult('red')"
        >
          Red
        </div>
      </div>

      <div
        v-if="cardHistory.length > 0"
        class="flex w-full my-4 overflow-x-auto md:w-2/3"
      >
        <div
          v-for="(card, index) in cardHistory"
          :key="index"
          class="flex-shrink-0"
        >
          <div
            class="p-2 m-1 rounded-xl"
            :class="card == 'red' ? 'bg-red-600' : 'bg-blue-600'"
          ></div>
        </div>
      </div>
    </div>

    <!-- ปุ่มเลือกผลล่าสุด -->
    <div class="my-6 space-x-4">
      <button
        @click="submitCardResult('red')"
        class="px-4 py-2 text-white bg-red-500 rounded hover:bg-red-600"
      >
        Red
      </button>
      <button
        @click="submitCardResult('blue')"
        class="px-4 py-2 text-white bg-blue-500 rounded hover:bg-blue-600"
      >
        Blue
      </button>
      <button
        @click="removeLastCard"
        :disabled="cardHistory.length === 0"
        class="px-4 py-2 text-white bg-yellow-500 rounded hover:bg-yellow-600 disabled:opacity-50"
      >
        ลบที่เพิ่มล่าสุด
      </button>
      <button
        @click="clearAll"
        :disabled="cardHistory.length === 0"
        class="px-4 py-2 text-white bg-gray-500 rounded hover:bg-gray-600 disabled:opacity-50"
      >
        Clear All
      </button>
    </div>

    <!-- แสดงผลการทำนายและเปอร์เซ็นต์การทายถูก -->
    <div v-if="prediction" class="mb-6">
      <h2 class="text-2xl font-semibold">Prediction for Next Card:</h2>
      <p class="text-lg">{{ prediction }}</p>
      <p class="text-lg font-medium">Accuracy: {{ accuracy }}%</p>
    </div>

    <!-- แสดงประวัติการกรอกผลลัพธ์ -->
    <div
      v-if="cardHistory.length > 0"
      class="mt-4 h-[600px] bg-slate-200 p-4 overflow-auto"
    >
      <h2 class="text-xl font-semibold">History</h2>
      <ul class="pl-5 space-y-1 list-disc">
        <li
          v-for="(card, index) in cardHistory
            .slice() // ทำสำเนาเพื่อไม่ให้เปลี่ยนแปลงข้อมูลต้นฉบับ
            .sort((a, b) => cardHistory.indexOf(b) - cardHistory.indexOf(a)) // เรียงจาก index มากไปน้อย
            .reverse()"
          :key="index"
          :class="{
            'text-lg p-2 rounded': true,
            'bg-green-200':
              index > 0 &&
              card === cardHistory[cardHistory.indexOf(card) + 1] &&
              card === correctPredictionsHistory[cardHistory.indexOf(card)],
          }"
        >
          {{ cardHistory.length - index }}. {{ card }}
          <!-- แสดง index จากมากไปน้อย -->
        </li>
      </ul>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed } from "vue";

const prediction = ref<string | null>(null);
const cardHistory = ref<string[]>([]);
const correctPredictions = ref(0);
const totalPredictions = ref(0);
const correctPredictionsHistory = ref<string[]>([]); // บันทึกผลทำนายที่ถูกต้อง

const accuracy = computed(() =>
  totalPredictions.value > 0
    ? ((correctPredictions.value / totalPredictions.value) * 100).toFixed(2)
    : "N/A"
);

// ฟังก์ชันสำหรับเพิ่มผลล่าสุด
const submitCardResult = (result: string) => {
  // ทำนายก่อนเพิ่มผลลัพธ์ใหม่
  const predictedCard = predictNextCard();

  // เพิ่มผลล่าสุดเข้าไปใน cardHistory
  cardHistory.value.push(result);

  // บันทึกผลลัพธ์ที่ทำนายถูก
  correctPredictionsHistory.value.push(predictedCard === result ? result : "");

  // ตรวจสอบว่าทำนายถูกหรือไม่
  if (predictedCard === result) {
    correctPredictions.value += 1;
  }

  // อัปเดตจำนวนการทำนายทั้งหมด
  totalPredictions.value += 1;

  // อัปเดตการทำนายใหม่
  prediction.value = predictNextCard();
};

// ฟังก์ชันสำหรับลบผลล่าสุดที่เพิ่มเข้ามา
const removeLastCard = () => {
  if (cardHistory.value.length === 0) return;

  const lastCard = cardHistory.value.pop();
  correctPredictionsHistory.value.pop(); // ลบผลทำนายที่ถูกต้อง

  // ตรวจสอบว่าผลลัพธ์ล่าสุดที่ลบออกนั้นทำนายถูกหรือไม่
  if (lastCard === prediction.value) {
    correctPredictions.value -= 1;
  }

  // ลดจำนวนการทำนายทั้งหมด
  totalPredictions.value -= 1;

  // อัปเดตการทำนายใหม่
  prediction.value = predictNextCard();

  // เลื่อน scroll ไปยังรายการล่าสุด
  scrollToLatestCard();
};

// ฟังก์ชันเลื่อน scroll ไปยังรายการล่าสุด
const scrollToLatestCard = () => {
  const container = document.querySelector(".overflow-x-auto");
  if (container) {
    container.scrollTo({ left: container.scrollWidth, behavior: "smooth" });
  }
};

// ฟังก์ชันสำหรับล้างข้อมูลทั้งหมด
const clearAll = () => {
  cardHistory.value = [];
  correctPredictionsHistory.value = [];
  prediction.value = null;
  correctPredictions.value = 0;
  totalPredictions.value = 0;
};

// ฟังก์ชันสำหรับสร้าง Transition Matrix
const buildTransitionMatrix = () => {
  const transitions: Record<string, Record<string, number>> = {
    red: { red: 0, blue: 0 },
    blue: { red: 0, blue: 0 },
  };

  for (let i = 0; i < cardHistory.value.length - 1; i++) {
    const current = cardHistory.value[i];
    const next = cardHistory.value[i + 1];
    transitions[current][next] += 1;
  }

  Object.keys(transitions).forEach((state) => {
    const total = transitions[state]["red"] + transitions[state]["blue"];
    if (total > 0) {
      transitions[state]["red"] /= total;
      transitions[state]["blue"] /= total;
    }
  });

  return transitions;
};

// ฟังก์ชันสำหรับทำนายไพ่ถัดไปโดยใช้ Markov Chain
const predictNextCard = (): string => {
  if (cardHistory.value.length < 2) return "No data yet";

  const lastCard = cardHistory.value[cardHistory.value.length - 1];
  const transitions = buildTransitionMatrix();

  const probabilityRed = transitions[lastCard]["red"];
  const probabilityBlue = transitions[lastCard]["blue"];

  return probabilityRed >= probabilityBlue ? "red" : "blue";
};
</script>

<style scoped>
button {
  padding: 10px;
  margin: 5px;
  font-size: 16px;
  cursor: pointer;
}
button:disabled {
  cursor: not-allowed;
  opacity: 0.5;
}
ul {
  list-style-type: none;
  padding: 0;
}
li {
  font-size: 16px;
}
</style>
