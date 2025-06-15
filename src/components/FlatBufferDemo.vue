<script setup lang="ts">
import { ref, reactive } from "vue";
import * as flatbuffers from "flatbuffers";
import { MyGame } from "../flatbuffers/person";

// 原始数据，模拟一个用户对象
const originalData = reactive({
  id: 1234567890123, // 注意：大整数，FlatBuffers 需要用 BigInt 处理
  name: "张伟",
  email: "zhang.wei@example.com",
  age: 32,
  height: 1.78,
  isVerified: true,
  skills: ["JavaScript", "TypeScript", "Vue", "Node.js"],
  location: {
    latitude: 31.2304,
    longitude: 121.4737,
  },
});

// const generateDataset = (count: number) => {
//   const dataset = [];
//   for (let i = 0; i < count; i++) {
//     dataset.push({
//       id: i + 1000000000000,
//       name: `User${i}`,
//       email: `user${i}@example.com`,
//       age: 20 + (i % 30),
//       height: 1.6 + (i % 40) / 100,
//       isVerified: i % 3 === 0,
//       skills: ["JS", "TS", `Skill${i % 10}`],
//       location: {
//         latitude: 30 + (i % 100) / 100,
//         longitude: 120 + (i % 100) / 100,
//       },
//     });
//   }
//   return dataset;
// };

// const originalDataList = generateDataset(1);

// 用于存储序列化和反序列化的结果
const serializedData = ref<Uint8Array | null>(null); // FlatBuffer 二进制数据
const deserializedData = ref<any>(null); // FlatBuffer 反序列化后的对象
const jsonSerialized = ref<string>(""); // JSON 字符串
const jsonDeserialized = ref<any>(null); // JSON 反序列化后的对象

// 性能统计数据
const performanceResults = reactive({
  serializeTime: 0, // FlatBuffer 序列化耗时
  deserializeTime: 0, // FlatBuffer 反序列化耗时
  jsonSerializeTime: 0, // JSON 序列化耗时
  jsonDeserializeTime: 0, // JSON 反序列化耗时
  flatbufferSize: 0, // FlatBuffer 数据大小
  jsonSize: 0, // JSON 数据大小
});

// 序列化为 FlatBuffer 二进制
const serializeToFlatBuffer = () => {
  const startTime = performance.now();

  // 创建 FlatBuffer 构建器
  const builder = new flatbuffers.Builder(1024);
  // 创建字符串字段
  const nameOffset = builder.createString(originalData.name);
  const emailOffset = builder.createString(originalData.email);

  // 创建技能字符串数组
  const skillsOffsets: flatbuffers.Offset[] = [];
  originalData.skills.forEach((skill) => {
    skillsOffsets.push(builder.createString(skill));
  });
  // 创建技能向量（注意顺序是否需要 reverse，视生成代码而定）
  const skillsVector = MyGame.Person.createSkillsVector(builder, skillsOffsets);

  // 创建地理位置结构体
  const locationOffset = MyGame.Location.createLocation(
    builder,
    originalData.location.latitude,
    originalData.location.longitude
  );

  // 构建 Person 对象
  MyGame.Person.startPerson(builder);
  MyGame.Person.addId(builder, BigInt(originalData.id)); // id 需转为 BigInt
  MyGame.Person.addName(builder, nameOffset);
  MyGame.Person.addEmail(builder, emailOffset);
  MyGame.Person.addAge(builder, originalData.age);
  MyGame.Person.addHeight(builder, originalData.height);
  MyGame.Person.addIsVerified(builder, originalData.isVerified);
  MyGame.Person.addSkills(builder, skillsVector);
  MyGame.Person.addLocation(builder, locationOffset);
  // 完成构建并获取数据
  const personOffset = MyGame.Person.endPerson(builder);
  builder.finish(personOffset);

  serializedData.value = builder.asUint8Array(); // 获取最终二进制数据
  performanceResults.serializeTime = performance.now() - startTime;
  performanceResults.flatbufferSize = serializedData.value.length;
};

// 从 FlatBuffer 二进制反序列化为对象
const deserializeFromFlatBuffer = () => {
  if (!serializedData.value) return;

  const startTime = performance.now();
  const buf = new flatbuffers.ByteBuffer(serializedData.value);
  const person = MyGame.Person.getRootAsPerson(buf);

  // 使用对象 API（unpack）简化访问
  const personObj = person.unpack();

  // 组装反序列化后的对象，便于展示
  deserializedData.value = {
    id: personObj.id,
    name: personObj.name,
    email: personObj.email,
    age: personObj.age,
    height: personObj.height,
    isVerified: personObj.isVerified,
    skills: personObj.skills || [],
    location: {
      latitude: personObj.location?.latitude || 0,
      longitude: personObj.location?.longitude || 0,
    },
  };

  performanceResults.deserializeTime = performance.now() - startTime;
};

// 序列化为 JSON 字符串
const serializeToJson = () => {
  const startTime = performance.now();
  jsonSerialized.value = JSON.stringify(originalData);
  performanceResults.jsonSerializeTime = performance.now() - startTime;
  performanceResults.jsonSize = new Blob([jsonSerialized.value]).size;
};

// 从 JSON 字符串反序列化为对象
const deserializeFromJson = () => {
  if (!jsonSerialized.value) return;

  const startTime = performance.now();
  jsonDeserialized.value = JSON.parse(jsonSerialized.value);
  performanceResults.jsonDeserializeTime = performance.now() - startTime;
};

// 格式化性能数值，微秒/毫秒自适应
const formatPerformance = (ms: number) => {
  return ms < 1 ? `${(ms * 1000).toFixed(2)} µs` : `${ms.toFixed(2)} ms`;
};

// 解决 BigInt 不能被 JSON.stringify 的问题，序列化时将 BigInt 转为字符串
function stringifyWithBigInt(obj: any) {
  return JSON.stringify(
    obj,
    (_key, value) => (typeof value === "bigint" ? value.toString() : value),
    2
  );
}
</script>

<template>
  <div class="container">
    <header class="header">
      <h1>Vue3 + TypeScript FlatBuffers 演示</h1>
      <p>展示高效二进制序列化与 JSON 的性能对比</p>
    </header>

    <main class="content">
      <section class="data-section">
        <h2>原始数据</h2>
        <pre class="data-box">{{ originalData }}</pre>
      </section>

      <div class="actions">
        <div class="action-group">
          <h3>FlatBuffers 操作</h3>
          <button @click="serializeToFlatBuffer" class="btn primary">
            序列化为 FlatBuffer
          </button>
          <button
            @click="deserializeFromFlatBuffer"
            class="btn success"
            :disabled="!serializedData"
          >
            从 FlatBuffer 反序列化
          </button>
        </div>

        <div class="action-group">
          <h3>JSON 操作</h3>
          <button @click="serializeToJson" class="btn primary">
            序列化为 JSON
          </button>
          <button
            @click="deserializeFromJson"
            class="btn success"
            :disabled="!jsonSerialized"
          >
            从 JSON 反序列化
          </button>
        </div>
      </div>

      <div class="results">
        <div class="result-card">
          <h3>FlatBuffer 序列化结果</h3>
          <div v-if="serializedData" class="result-content">
            <p>大小: {{ performanceResults.flatbufferSize }} 字节</p>
            <div class="hex-view">
              <span
                v-for="(byte, index) in serializedData"
                :key="index"
                class="hex-byte"
              >
                {{ byte.toString(16).padStart(2, "0") }}
              </span>
            </div>
          </div>
          <div v-else class="placeholder">
            点击"序列化为 FlatBuffer"按钮生成数据
          </div>
        </div>

        <div class="result-card">
          <h3>FlatBuffer 反序列化结果</h3>
          <pre v-if="deserializedData" class="data-box">{{
            stringifyWithBigInt(deserializedData)
          }}</pre>
          <div v-else class="placeholder">
            序列化后点击"从 FlatBuffer 反序列化"按钮
          </div>
        </div>

        <div class="result-card">
          <h3>JSON 序列化结果</h3>
          <div v-if="jsonSerialized" class="result-content">
            <p>大小: {{ performanceResults.jsonSize }} 字节</p>
            <pre class="json-view">{{ jsonSerialized }}</pre>
          </div>
          <div v-else class="placeholder">点击"序列化为 JSON"按钮生成数据</div>
        </div>

        <div class="result-card">
          <h3>JSON 反序列化结果</h3>
          <pre v-if="jsonDeserialized" class="data-box">{{
            jsonDeserialized
          }}</pre>
          <div v-else class="placeholder">
            序列化后点击"从 JSON 反序列化"按钮
          </div>
        </div>
      </div>

      <section class="performance-section">
        <h2>性能对比</h2>
        <div class="performance-grid">
          <div class="metric">
            <div class="metric-value">
              {{ formatPerformance(performanceResults.serializeTime) }}
            </div>
            <div class="metric-label">FlatBuffer 序列化时间</div>
          </div>
          <div class="metric">
            <div class="metric-value">
              {{ formatPerformance(performanceResults.deserializeTime) }}
            </div>
            <div class="metric-label">FlatBuffer 反序列化时间</div>
          </div>
          <div class="metric">
            <div class="metric-value">
              {{ formatPerformance(performanceResults.jsonSerializeTime) }}
            </div>
            <div class="metric-label">JSON 序列化时间</div>
          </div>
          <div class="metric">
            <div class="metric-value">
              {{ formatPerformance(performanceResults.jsonDeserializeTime) }}
            </div>
            <div class="metric-label">JSON 反序列化时间</div>
          </div>
          <div class="metric">
            <div class="metric-value">
              {{ performanceResults.flatbufferSize }} 字节
            </div>
            <div class="metric-label">FlatBuffer 大小</div>
          </div>
          <div class="metric">
            <div class="metric-value">
              {{ performanceResults.jsonSize }} 字节
            </div>
            <div class="metric-label">JSON 大小</div>
          </div>
        </div>

        <div class="performance-summary">
          <div class="summary-item">
            <span class="summary-label">FlatBuffer 大小减少:</span>
            <span class="summary-value" v-if="performanceResults.jsonSize > 0">
              {{
                Math.round(
                  (1 -
                    performanceResults.flatbufferSize /
                      performanceResults.jsonSize) *
                    100
                )
              }}%
            </span>
          </div>
          <div class="summary-item">
            <span class="summary-label">反序列化速度提升:</span>
            <span
              class="summary-value"
              v-if="
                performanceResults.jsonDeserializeTime > 0 &&
                performanceResults.deserializeTime > 0
              "
            >
              {{
                Math.round(
                  performanceResults.jsonDeserializeTime /
                    performanceResults.deserializeTime
                )
              }}x
            </span>
          </div>
        </div>
      </section>
    </main>

    <footer class="footer">
      <p>FlatBuffers 演示应用 | Vue3 + TypeScript</p>
      <p>FlatBuffers 提供高效的二进制序列化，特别适合高性能应用</p>
    </footer>
  </div>
</template>

<style scoped>
.container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
  font-family: "Segoe UI", Tahoma, Geneva, Verdana, sans-serif;
  color: #333;
  background-color: #f8f9fa;
  min-height: 100vh;
}

.header {
  text-align: center;
  margin-bottom: 30px;
  padding: 20px;
  background: linear-gradient(135deg, #6a11cb 0%, #2575fc 100%);
  color: white;
  border-radius: 10px;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.1);
}

.header h1 {
  margin: 0;
  font-size: 2.2rem;
}

.header p {
  margin: 10px 0 0;
  opacity: 0.9;
  font-size: 1.1rem;
}

.content {
  background: white;
  border-radius: 10px;
  padding: 25px;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.05);
}

.data-section {
  margin-bottom: 25px;
}

.data-section h2 {
  color: #2c3e50;
  border-bottom: 2px solid #eee;
  padding-bottom: 10px;
}

.data-box {
  background: #f8f9fa;
  border: 1px solid #e9ecef;
  border-radius: 8px;
  padding: 15px;
  max-height: 200px;
  overflow: auto;
  font-size: 0.9rem;
  line-height: 1.5;
}

.actions {
  display: flex;
  gap: 20px;
  margin-bottom: 30px;
  flex-wrap: wrap;
}

.action-group {
  flex: 1;
  min-width: 280px;
  background: #f8f9fa;
  padding: 20px;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.05);
}

.action-group h3 {
  margin-top: 0;
  color: #2c3e50;
  border-bottom: 1px solid #e9ecef;
  padding-bottom: 10px;
}

.btn {
  padding: 10px 15px;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-weight: 600;
  transition: all 0.3s ease;
  margin-right: 10px;
  margin-bottom: 10px;
}

.btn.primary {
  background: #3498db;
  color: white;
}

.btn.success {
  background: #2ecc71;
  color: white;
}

.btn:disabled {
  background: #95a5a6;
  cursor: not-allowed;
}

.btn:not(:disabled):hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
}

.results {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 20px;
  margin-bottom: 30px;
}

.result-card {
  background: #f8f9fa;
  border-radius: 8px;
  padding: 20px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.05);
}

.result-card h3 {
  margin-top: 0;
  color: #2c3e50;
  border-bottom: 1px solid #e9ecef;
  padding-bottom: 10px;
}

.hex-view {
  display: flex;
  flex-wrap: wrap;
  gap: 4px;
  font-family: monospace;
  font-size: 0.8rem;
}

.hex-byte {
  background: #e9ecef;
  padding: 2px 4px;
  border-radius: 3px;
}

.json-view {
  background: #f8f9fa;
  border: 1px solid #e9ecef;
  border-radius: 5px;
  padding: 10px;
  max-height: 150px;
  overflow: auto;
  font-size: 0.85rem;
}

.placeholder {
  text-align: center;
  padding: 30px;
  color: #6c757d;
  background: #e9ecef;
  border-radius: 5px;
}

.performance-section {
  margin-top: 30px;
}

.performance-section h2 {
  color: #2c3e50;
  border-bottom: 2px solid #eee;
  padding-bottom: 10px;
}

.performance-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 20px;
  margin-top: 20px;
}

.metric {
  background: white;
  border-radius: 8px;
  padding: 20px;
  text-align: center;
  box-shadow: 0 4px 10px rgba(0, 0, 0, 0.05);
  border: 1px solid #e9ecef;
  transition: transform 0.3s ease;
}

.metric:hover {
  transform: translateY(-5px);
  box-shadow: 0 6px 15px rgba(0, 0, 0, 0.1);
}

.metric-value {
  font-size: 1.5rem;
  font-weight: 700;
  color: #2980b9;
  margin-bottom: 8px;
}

.metric-label {
  font-size: 0.95rem;
  color: #7f8c8d;
}

.performance-summary {
  display: flex;
  justify-content: center;
  gap: 40px;
  margin-top: 30px;
  padding: 20px;
  background: linear-gradient(to right, #e3f2fd, #bbdefb);
  border-radius: 8px;
}

.summary-item {
  text-align: center;
}

.summary-label {
  display: block;
  font-size: 1.1rem;
  font-weight: 600;
  color: #2c3e50;
}

.summary-value {
  display: block;
  font-size: 1.8rem;
  font-weight: 700;
  color: #e74c3c;
}

.footer {
  text-align: center;
  margin-top: 30px;
  padding: 20px;
  color: #7f8c8d;
  font-size: 0.9rem;
}
</style>
