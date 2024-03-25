<template>
  <h1 style="color: black;text-align: center;">
    AI Ark Graph generator
  </h1>
  <el-container>
    <el-card class="form">
      <h3>
        Please upload a file to get started
      </h3>

      <el-upload accept="xlsx,xls" :on-change="handleChange" :auto-upload="false">
        <el-button type="primary">Click to upload</el-button>
        <template #tip>
          <div class="el-upload__tip">
            xls/xlsx files
          </div>
        </template>
      </el-upload>

        <el-form label-position="top" v-if="fileUploaded">
          <el-form-item label="Question">
            <el-input label="Question" rows="4" type="textarea" v-model="message" placeholder="Please input.." />
          </el-form-item>

          <el-button :loading="loading" type="primary"  @click="askQuestion">Generate</el-button>


        </el-form>

    </el-card>

  </el-container>
  <br>
  <el-card class="chart-wrapper" v-if="Object.keys(chartData).length">
    <el-radio-group v-model="chartType" class="ml-4">
      <el-radio value="bar" size="large">Bar chart</el-radio>
      <el-radio value="pie" size="large">Pie chart</el-radio>
    </el-radio-group>
    <canvas v-if="!loading" ref="chartRef" id="myChart"></canvas>

    <template v-else>
        <el-skeleton />
        <br />
        <el-skeleton style="--el-skeleton-circle-size: 100px">
          <template #template>
            <el-skeleton-item variant="circle" />
          </template>
        </el-skeleton>
    </template>
  </el-card>
</template>

<script setup lang="ts">
import OpenAI from "openai";
import {reactive, ref, watchEffect} from "vue";
import {Chart, ChartData} from "chart.js/auto";
import {QUESTION_FILLER} from "./constants/questionFiller.ts";
import {read, utils} from "xlsx";

const message = ref('');
const chartRef = ref(null);
const chartType = ref<'pie' | 'bar'>('bar');
const chartData = reactive<Partial<ChartData>>({})
const fileUploaded = ref(false);
const loading = ref(false);

const messages = ref<{role: 'user' | 'assistant', content: string}[]>([])

let myChart: Chart;

const openai = new OpenAI({
  apiKey: import.meta.env.VITE_OPENAI_API_KEY,
  dangerouslyAllowBrowser: true,
});

watchEffect(() => {
  if(!chartRef.value) {
    return;
  }

  if(myChart instanceof Chart) {
    myChart.destroy();
  }

  myChart = new Chart(chartRef.value, {
    type: chartType.value,
    data: chartData,
  });
})

function addUserMessage(message: string) {
  messages.value.push({role: 'user', content: message});
}

function addAssistantMessage(message: string) {
  messages.value.push({role: 'assistant', content: message});
}

async function handleChange({raw}: {raw: File}) {
  fileUploaded.value = true;

  const workbook = read(await raw.arrayBuffer());
  utils.sheet_to_json(workbook.Sheets[workbook.SheetNames[0]] );

  const sourceData = JSON.stringify(utils.sheet_to_json(workbook.Sheets[workbook.SheetNames[0]]));

  messages.value = [
    {role: 'user', content: sourceData.concat('\n', 'This is the data based on i want your answers for my next questions')},
    {role: 'assistant', content: 'Sure, go ahead and ask your questions based on the provided data. I\'ll do my best to help you with them.'}
  ]
}

async function askQuestion() {
  loading.value = true;
  addUserMessage(message.value.concat(`\n${QUESTION_FILLER}`));

  const chatCompletion = await openai.chat.completions.create({
    messages: messages.value,
    model: "gpt-3.5-turbo",
  });

  const answer = chatCompletion.choices[0].message.content as string;

  addAssistantMessage(answer);

  console.log(messages.value);
  console.log(JSON.parse(answer), 'answer')

  Object.assign(chartData, JSON.parse(answer));

  loading.value = false;
}

</script>



<style lang="scss">
.el-main {
  text-align: center;
}
.el-container {
  justify-content: center;
}


.form {

  min-width: 280px;
}

.el-card__body {
  display: flex;
  flex-direction: column;
}

.el-textarea, .el-upload, .el-button--primary, .chart-wrapper  {
  width: 100%;
}

</style>
