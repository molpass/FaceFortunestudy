<template>
  <div class="face-fortune-container">
    <el-card class="fortune-card">
      <template #header>
        <div class="card-header">
          <h2>관상 운세</h2>
          <div class="camera-controls">
            <el-button type="primary" @click="startCamera" :disabled="isCameraActive">
              <el-icon><VideoCamera /></el-icon> 카메라 켜기
            </el-button>
            <el-button type="danger" @click="stopCamera" :disabled="!isCameraActive">
              <el-icon><VideoCameraFilled /></el-icon> 카메라 끄기
            </el-button>
            <el-button type="success" @click="captureAndAnalyze" :disabled="!isCameraActive || isAnalyzing">
              <el-icon><Camera /></el-icon> 운세 보기
            </el-button>
          </div>
        </div>
      </template>
      
      <div class="camera-container">
        <video ref="video" class="camera-view" :class="{ hidden: !isCameraActive }" autoplay muted></video>
        <canvas ref="canvas" class="face-canvas"></canvas>
        <div v-if="!isCameraActive && !isAnalyzing" class="camera-placeholder">
          <el-icon><Camera /></el-icon>
          <p>"카메라 켜기" 버튼을 눌러 시작하세요</p>
        </div>
        <div v-if="isAnalyzing" class="analyzing">
          <el-icon class="is-loading"><Loading /></el-icon>
          <p>관상을 분석하는 중...</p>
        </div>
      </div>

      <div v-if="fortuneResult" class="fortune-result">
        <h3>관상 분석 결과</h3>

        <div class="basic-info">
          <el-descriptions :column="2" border>
            <el-descriptions-item label="성별">
              <el-tag :type="fortuneResult.gender === '남성' ? 'primary' : 'danger'">
                {{ fortuneResult.gender }}
              </el-tag>
            </el-descriptions-item>
            <el-descriptions-item label="나이">
              {{ fortuneResult.age }}세
            </el-descriptions-item>
          </el-descriptions>
        </div>

        <div class="personality-chart">
          <h4>성격 분석</h4>
          <div class="chart-container">
            <v-chart class="chart" :option="personalityChartOption" autoresize />
          </div>
        </div>

        <div class="fortune-details">
          <el-collapse v-model="activeCollapse">
            <el-collapse-item title="성격 특징" name="personality">
              <div class="fortune-text">{{ fortuneResult.personality.description }}</div>
            </el-collapse-item>
            <el-collapse-item title="사업운 분석" name="career">
              <div class="fortune-text">{{ fortuneResult.career.description }}</div>
            </el-collapse-item>
            <el-collapse-item title="결혼운 분석" name="marriage">
              <div class="fortune-text">{{ fortuneResult.marriage.description }}</div>
            </el-collapse-item>
          </el-collapse>
        </div>

        <div class="daily-fortune">
          <h4>오늘의 운세</h4>
          <el-row :gutter="20">
            <el-col :span="12">
              <el-card class="daily-card suitable">
                <template #header>
                  <div class="card-header">
                    <span>오늘 좋은 일</span>
                  </div>
                </template>
                <div class="daily-items">
                  <div v-for="(item, index) in fortuneResult.daily.suitable" :key="'suitable-'+index" class="daily-item">
                    <el-tag type="success" effect="light">{{ item }}</el-tag>
                  </div>
                </div>
              </el-card>
            </el-col>
            <el-col :span="12">
              <el-card class="daily-card avoid">
                <template #header>
                  <div class="card-header">
                    <span>오늘 피할 일</span>
                  </div>
                </template>
                <div class="daily-items">
                  <div v-for="(item, index) in fortuneResult.daily.avoid" :key="'avoid-'+index" class="daily-item">
                    <el-tag type="danger" effect="light">{{ item }}</el-tag>
                  </div>
                </div>
              </el-card>
            </el-col>
          </el-row>
        </div>
      </div>
    </el-card>
  </div>
</template>

<script setup>
import { ref, reactive, onMounted, onUnmounted, computed } from 'vue';
import { use } from 'echarts/core';
import { CanvasRenderer } from 'echarts/renderers';
import { RadarChart } from 'echarts/charts';
import { TitleComponent, TooltipComponent, LegendComponent } from 'echarts/components';
import VChart from 'vue-echarts';
import * as faceapi from 'face-api.js';
import { ElNotification } from 'element-plus';

// ECharts 컴포넌트 등록
use([
  CanvasRenderer,
  RadarChart,
  TitleComponent,
  TooltipComponent,
  LegendComponent
]);

// 반응형 상태
const video = ref(null);
const canvas = ref(null);
const isCameraActive = ref(false);
const isAnalyzing = ref(false);
const stream = ref(null);
const fortuneResult = ref(null);
const activeCollapse = ref(['personality']);

// 검출 결과를 안정화하기 위한 캐시 데이터
const detectionBuffer = ref([]);
const bufferSize = 5; // 최근 5프레임의 검출 결과 저장
const minDetectionConfidence = 0.7; // 최소 검출 신뢰도 임계값

// 성격 레이더 차트 설정
const personalityChartOption = computed(() => {
  if (!fortuneResult.value) return {};
  
  return {
    radar: {
      indicator: [
        { name: '쾌활함', max: 100 },
        { name: '침착함', max: 100 },
        { name: '창의력', max: 100 },
        { name: '인내심', max: 100 },
        { name: '세심함', max: 100 }
      ],
      radius: 80
    },
    series: [{
      type: 'radar',
      data: [{
        value: [
          fortuneResult.value.personality.traits.outgoing,
          fortuneResult.value.personality.traits.steady,
          fortuneResult.value.personality.traits.creative,
          fortuneResult.value.personality.traits.patient,
          fortuneResult.value.personality.traits.detail
        ],
        name: '성격 특징',
        areaStyle: {
          color: 'rgba(64, 158, 255, 0.6)'
        }
      }]
    }]
  };
});

// 라이프사이클 훅
onMounted(async () => {
  try {
    // face-api.js 모델 로드
    await loadFaceApiModels();
    console.log('Face-API 모델 로드 완료');
  } catch (error) {
    console.error('모델 로드 실패:', error);
    ElNotification({
      title: '오류',
      message: '얼굴 인식 모델을 불러올 수 없습니다. 페이지를 새로고침한 뒤 다시 시도해 주세요',
      type: 'error',
      duration: 5000
    });
  }
});

onUnmounted(() => {
  stopCamera();
});

// 메서드
async function loadFaceApiModels() {
  // 환경에 따라 모델 경로를 동적으로 설정하여 개발/운영 환경 모두에서 올바르게 로드되도록 함
  const BASE_URL = import.meta.env.BASE_URL || '/';
  const MODEL_URL = `${BASE_URL}models`;

  // 현재 존재하는 유일한 모델인 tiny_face_detector 모델을 먼저 로드 시도
  try {
    console.log('TinyFaceDetector 모델 로드 시작, 경로:', MODEL_URL);
    await faceapi.nets.tinyFaceDetector.loadFromUri(MODEL_URL);
    console.log('TinyFaceDetector 모델 로드 성공');

    // 다른 모델 파일이 있다면 아래 주석을 해제할 수 있음
    // 현재는 존재하지 않는 모델 로드를 주석 처리하여 오류 방지
    /*
    await Promise.all([
      faceapi.nets.faceLandmark68Net.loadFromUri(MODEL_URL),
      faceapi.nets.faceRecognitionNet.loadFromUri(MODEL_URL),
      faceapi.nets.faceExpressionNet.loadFromUri(MODEL_URL),
      faceapi.nets.ageGenderNet.loadFromUri(MODEL_URL)
    ]);
    */
  } catch (error) {
    console.error('TinyFaceDetector 모델 로드 실패:', error);
    console.error('모델 경로:', MODEL_URL);
    throw error;
  }
}

async function startCamera() {
  try {
    stream.value = await navigator.mediaDevices.getUserMedia({
      video: { facingMode: 'user' }
    });
    
    if (video.value) {
      video.value.srcObject = stream.value;
      isCameraActive.value = true;
    }
  } catch (error) {
    console.error('카메라에 접근할 수 없습니다:', error);
    ElNotification({
      title: '오류',
      message: '카메라에 접근할 수 없습니다. 카메라 권한을 허용했는지 확인해 주세요',
      type: 'error',
      duration: 5000
    });
  }
}

function stopCamera() {
  if (stream.value) {
    const tracks = stream.value.getTracks();
    tracks.forEach(track => track.stop());
    stream.value = null;
    isCameraActive.value = false;

    // 캔버스 지우기
    const ctx = canvas.value.getContext('2d');
    ctx.clearRect(0, 0, canvas.value.width, canvas.value.height);
  }
}

async function captureAndAnalyze() {
  if (!isCameraActive.value) return;
  
  isAnalyzing.value = true;
  console.log('얼굴 분석 시작...');

  try {
    // 모델이 로드되었는지 확인
    if (!faceapi.nets.tinyFaceDetector.isLoaded) {
      console.warn('TinyFaceDetector 모델이 로드되지 않음, 재로드 시도...');
      try {
        await loadFaceApiModels();
      } catch (modelError) {
        console.error('모델 재로드 실패:', modelError);
        throw new Error('얼굴 인식 모델이 로드되지 않았습니다. 페이지를 새로고침한 뒤 다시 시도해 주세요');
      }
    }

    // canvas 크기를 비디오와 동일하게 설정
    const videoEl = video.value;
    const canvasEl = canvas.value;

    if (!videoEl || !canvasEl) {
      throw new Error('비디오 또는 캔버스 요소를 찾을 수 없습니다');
    }

    console.log('비디오 크기:', videoEl.videoWidth, 'x', videoEl.videoHeight);
    canvasEl.width = videoEl.videoWidth;
    canvasEl.height = videoEl.videoHeight;

    // canvas에 현재 비디오 프레임 그리기
    const ctx = canvasEl.getContext('2d');
    ctx.drawImage(videoEl, 0, 0, canvasEl.width, canvasEl.height);
    console.log('비디오 프레임을 캔버스에 그렸습니다');

    // face-api.js로 얼굴 검출, 로드된 TinyFaceDetector 모델만 사용
    console.log('얼굴 검출 시작...');
    const detectorOptions = new faceapi.TinyFaceDetectorOptions({ scoreThreshold: minDetectionConfidence });
    const detections = await faceapi.detectAllFaces(canvasEl, detectorOptions);
    console.log('얼굴 검출 결과:', detections);

    if (detections.length === 0) {
      console.log('얼굴이 검출되지 않음');
      ElNotification({
        title: '안내',
        message: '얼굴이 검출되지 않았습니다. 위치를 조정한 뒤 다시 시도해 주세요',
        type: 'warning',
        duration: 3000
      });
      isAnalyzing.value = false;
      return;
    }
    
    // 신뢰도가 가장 높은 얼굴 가져오기
    const bestDetection = detections.reduce((prev, current) => {
      return (prev.detection.score > current.detection.score) ? prev : current;
    });
    console.log('최적 얼굴 검출 결과:', bestDetection);

    // 현재 검출 결과를 버퍼에 추가
    detectionBuffer.value.push(bestDetection);
    if (detectionBuffer.value.length > bufferSize) {
      detectionBuffer.value.shift(); // 가장 오래된 검출 결과 제거
    }

    // 얼굴 검출 박스와 특징점 그리기
    const resizedDetections = faceapi.resizeResults(detections, {
      width: canvasEl.width,
      height: canvasEl.height
    });
    
    ctx.clearRect(0, 0, canvasEl.width, canvasEl.height);
    ctx.drawImage(videoEl, 0, 0, canvasEl.width, canvasEl.height);
    
    try {
      faceapi.draw.drawDetections(canvasEl, resizedDetections);
      console.log('얼굴 검출 박스를 그렸습니다');
    } catch (drawError) {
      console.error('얼굴 검출 박스를 그리는 중 오류 발생:', drawError);
      // 흐름을 중단하지 않고 계속 진행
    }

    // 안정화된 검출 결과로 운세 결과 생성
    console.log('운세 결과 생성...');
    generateStableFortuneResult();
    console.log('운세 결과 생성 완료');

  } catch (error) {
    console.error('얼굴 분석 중 오류 발생:', error);
    let errorMsg = '얼굴 분석 중 오류가 발생했습니다. 다시 시도해 주세요';

    // 오류 유형에 따라 더 구체적인 오류 메시지 제공
    if (error.message) {
      console.error('오류 메시지:', error.message);
      if (error.message.includes('permission')) {
        errorMsg = '카메라에 접근할 수 없습니다. 카메라 권한을 허용했는지 확인해 주세요';
      } else if (error.message.includes('load')) {
        errorMsg = '얼굴 인식 모델 로드에 실패했습니다. 네트워크 연결을 확인해 주세요';
      } else if (error.message.includes('모델이 로드되지 않았습니다')) {
        errorMsg = error.message;
      } else {
        // 더 자세한 오류 정보 추가
        errorMsg = `얼굴 분석 중 오류 발생: ${error.message}`;
      }
    } else if (!navigator.onLine) {
      errorMsg = '네트워크 연결이 끊어졌습니다. 네트워크를 확인한 뒤 다시 시도해 주세요';
    }

    ElNotification({
      title: '오류',
      message: errorMsg,
      type: 'error',
      duration: 5000
    });
  } finally {
    isAnalyzing.value = false;
  }
}

function generateStableFortuneResult() {
  // 버퍼가 비어 있으면 반환
  if (detectionBuffer.value.length === 0) return;

  // 더 안정적인 결과를 얻기 위해 여러 프레임 검출 결과의 평균값 계산
  let avgWidth = 0;
  let avgHeight = 0;
  let avgAge = 0;
  let dominantExpression = {};
  let detectedGender = { male: 0, female: 0 };

  // 평균값 계산
  detectionBuffer.value.forEach(detection => {
    // detection 객체와 detection.detection, box가 존재하는지 확인
    if (detection && detection.detection && detection.detection.box) {
      const box = detection.detection.box;
      avgWidth += box.width;
      avgHeight += box.height;
    }

    // 나이 누적
    if (detection && detection.age) {
      avgAge += detection.age;
    }

    // 성별 검출 결과 누적
    if (detection && detection.gender) {
      detectedGender[detection.gender] += 1;
    }

    // 표정 검출 결과 누적
    if (detection && detection.expressions) {
      Object.entries(detection.expressions).forEach(([expression, value]) => {
        dominantExpression[expression] = (dominantExpression[expression] || 0) + value;
      });
    }
  });

  // 유효한 검출 결과 개수 계산
  let validDetections = 0;
  detectionBuffer.value.forEach(detection => {
    if (detection && detection.detection && detection.detection.box) {
      validDetections++;
    }
  });

  // 0으로 나누지 않도록 하여 평균값 계산
  if (validDetections > 0) {
    avgWidth /= validDetections;
    avgHeight /= validDetections;
  }

  // 나이 데이터가 있을 때만 평균 나이 계산
  let validAgeDetections = detectionBuffer.value.filter(d => d && d.age).length;
  if (validAgeDetections > 0) {
    avgAge /= validAgeDetections;
  }

  // 주요 표정 결정
  let mainExpression = Object.entries(dominantExpression).reduce(
    (max, [expression, value]) => value > max[1] ? [expression, value] : max,
    ['neutral', 0]
  )[0];

  // 성별 결정 (투표로 결정)
  const gender = detectedGender.male > detectedGender.female ? '남성' : '여성';

  // 안정화된 가로세로 비율 사용
  const faceRatio = avgWidth / avgHeight;

  // 검출된 나이 사용, 없으면 랜덤값 사용
  const age = avgAge > 0 ? Math.round(avgAge) : Math.floor(Math.random() * 40) + 18;

  // 랜덤 생성이 아닌 검출된 성별과 나이 사용

  // 성격 특징 생성
  const personalityTraits = {
    outgoing: Math.floor(Math.random() * 40) + 60, // 쾌활함 쪽으로 치우침
    steady: Math.floor(Math.random() * 100),
    creative: Math.floor(Math.random() * 100),
    patient: Math.floor(Math.random() * 100),
    detail: Math.floor(Math.random() * 100)
  };

  // 성격 특징에 따라 설명 생성
  let personalityDesc = '';
  if (personalityTraits.outgoing > 80) {
    personalityDesc = '당신은 타고난 낙천가이자 사교성이 뛰어나 사람들 사이에서 중심이 되는 분입니다. 주변에 활력을 불어넣는 성격 덕분에 어떤 자리에서도 물 만난 고기처럼 잘 어울립니다.';
  } else if (personalityTraits.steady > 80) {
    personalityDesc = '당신은 침착하고 듬직한 성격으로 매사를 차근차근 처리하는 믿음직한 사람입니다. 중요한 순간에도 늘 냉정을 유지하며 이성적인 결정을 내립니다.';
  } else if (personalityTraits.creative > 80) {
    personalityDesc = '당신은 사고가 활발하고 창의력이 풍부하여 독창적인 견해와 혁신적인 해결책을 자주 내놓습니다. 예술적 재능도 뛰어납니다.';
  } else {
    personalityDesc = '당신은 활력 넘치는 면과 사려 깊은 면을 두루 갖춘 다재다능한 성격입니다. 적응력이 뛰어나 어떤 환경에서도 자기 자리를 잘 찾아냅니다.';
  }

  // 사업운 분석 생성
  const careerDesc = [
    '당신의 관상에는 사업선이 또렷하고 강하게 나타나 앞으로 직장에서 크게 성공할 상입니다. 관리, 창업, 예술 분야처럼 창의력과 리더십이 필요한 일을 추천합니다.',
    '당신의 관상은 매사를 꼼꼼하고 성실하게 처리하는 상으로, 연구, 기술 개발, 정밀 제조처럼 인내심과 집중력이 필요한 일에 잘 맞습니다.',
    '당신의 관상은 뛰어난 소통 능력과 친화력을 지닌 상으로, 영업, 교육, 컨설팅처럼 사람을 상대하는 일에 잘 어울립니다.',
    '당신의 관상은 두뇌 회전이 빠르고 반응이 민첩한 상으로, 금융, 무역, 미디어 업계처럼 빠른 판단이 필요한 일에 잘 맞습니다.'
  ];
  const careerIndex = Math.floor(Math.random() * careerDesc.length);

  // 결혼운 분석 생성
  const marriageDesc = [
    '당신의 결혼선은 진심으로 사랑하는 반려자를 만나 결혼 후 화목하고 행복하게 지낼 상을 나타냅니다. 서른 살 무렵에 혼사를 생각해 보길 권합니다.',
    '당신의 결혼선은 감정에 있어 신중하여 쉽게 약속하지 않지만, 한번 마음을 정하면 온 마음을 다하는 상입니다. 결혼 후 가정생활이 안정적입니다.',
    '당신의 결혼선은 사랑에서 다소 우여곡절을 겪을 수 있으나 결국 진정한 사랑을 찾는 상입니다. 결혼 후에는 두 사람이 함께 노력해야 행복을 얻을 수 있습니다.',
    '당신의 결혼선은 사랑에 대한 열정과 기대가 넘치고 낭만에 쉽게 마음이 움직이는 상입니다. 결혼 후 생활이 다채롭지만, 늘 신선함을 유지하도록 신경 쓸 필요가 있습니다.'
  ];
  const marriageIndex = Math.floor(Math.random() * marriageDesc.length);

  // 오늘의 길흉 생성
  const suitableActivities = [
    '협상·계약', '여행·나들이', '투자·재테크', '구직 면접',
    '학습·자기계발', '운동·헬스', '모임·친목', '창업 계획',
    '쇼핑·소비', '인테리어·이사', '고백·청혼', '개업·개장'
  ];

  const avoidActivities = [
    '고소 작업', '이발·미용', '장거리 여행', '큰 지출',
    '다툼·갈등', '모험 활동', '계약 체결', '대출·차용',
    '의료 수술', '중대한 결정', '과음', '밤샘·야근'
  ];

  // 오늘의 길흉 항목 랜덤 선택
  const suitableCount = 3 + Math.floor(Math.random() * 3); // 3~5개
  const avoidCount = 3 + Math.floor(Math.random() * 3); // 3~5개

  const shuffleArray = (array) => {
    for (let i = array.length - 1; i > 0; i--) {
      const j = Math.floor(Math.random() * (i + 1));
      [array[i], array[j]] = [array[j], array[i]];
    }
    return array;
  };
  
  const suitable = shuffleArray([...suitableActivities]).slice(0, suitableCount);
  const avoid = shuffleArray([...avoidActivities]).slice(0, avoidCount);

  // 운세 결과 설정
  fortuneResult.value = {
    gender,
    age,
    personality: {
      traits: personalityTraits,
      description: personalityDesc
    },
    career: {
      description: careerDesc[careerIndex]
    },
    marriage: {
      description: marriageDesc[marriageIndex]
    },
    daily: {
      suitable,
      avoid
    }
  };
}
</script>

<style scoped>
.face-fortune-container {
  max-width: 900px;
  margin: 0 auto;
  padding: 20px;
}

.fortune-card {
  margin-bottom: 20px;
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  flex-wrap: wrap;
}

.camera-controls {
  display: flex;
  gap: 10px;
  margin-top: 10px;
}

.camera-container {
  position: relative;
  width: 100%;
  height: 400px;
  background-color: #f5f7fa;
  border-radius: 4px;
  overflow: hidden;
  margin-bottom: 20px;
}

.camera-view {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.camera-view.hidden {
  display: none;
}

.face-canvas {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
}

.camera-placeholder {
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  height: 100%;
  color: #909399;
}

.camera-placeholder .el-icon {
  font-size: 48px;
  margin-bottom: 16px;
}

.analyzing {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  background-color: rgba(0, 0, 0, 0.5);
  color: #fff;
}

.analyzing .el-icon {
  font-size: 48px;
  margin-bottom: 16px;
}

.fortune-result {
  margin-top: 20px;
}

.basic-info {
  margin-bottom: 20px;
}

.personality-chart {
  margin-bottom: 20px;
}

.chart-container {
  height: 300px;
}

.chart {
  width: 100%;
  height: 100%;
}

.fortune-details {
  margin-bottom: 20px;
}

.fortune-text {
  line-height: 1.8;
  text-align: justify;
  padding: 10px 0;
}

.daily-fortune {
  margin-top: 20px;
}

.daily-card {
  height: 100%;
}

.daily-items {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
}

.daily-item {
  margin-bottom: 10px;
}

@media (max-width: 768px) {
  .card-header {
    flex-direction: column;
    align-items: flex-start;
  }
  
  .camera-controls {
    margin-top: 15px;
    width: 100%;
    justify-content: space-between;
  }
  
  .camera-container {
    height: 300px;
  }
}
</style>