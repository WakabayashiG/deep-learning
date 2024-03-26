<template>
  <div>
    <a-row>

    </a-row>

    <a-row>

      <a-col :span="16">
        <a-badge status="processing" text="Step 1" />

        <a-card :bordered="false">
          <a-form layout="inline">
            <a-form-item label="Number of Inputs (Di)">
              <a-input-number v-model.number="networkParams.Di" @change="reset" />
            </a-form-item>
            <a-form-item label="Number of Hidden Layers (D)">
              <a-input-number v-model.number="networkParams.D" @change="reset"/>
            </a-form-item>
            <a-form-item label="Number of Outputs (Do)">
              <a-input-number v-model.number="networkParams.Do" @change="initializeNetwork()" />
            </a-form-item>

            <a-form-item>
              <a-button type="primary"  @click="calculateOutput">Calculate Output</a-button>
            </a-form-item>
          </a-form>
        </a-card>

        <br>

        <a-badge status="processing" text="Step 2" />
        <a-card>
          <a-form layout="inline">
            <div v-for="(input, index) in inputs" :key="'input-' + index">
              <a-form-item :label="'X' + (index + 1)">
                <a-input-number v-model.number="inputs[index]" />
              </a-form-item>
            </div>
          </a-form>


          <a-form layout="inline">
            <div v-for="(unit, index) in hiddenUnits" :key="'hidden-layer-' + index">
              <a-form-item :label="'D' + (index) + ' (Hidden Layer ' + (index) + ') '">
                <a-input-number v-model.number="hiddenUnits[index]" @change="initializeNetwork()" />
              </a-form-item>
            </div>
          </a-form>
        </a-card>

        <br>

        <a-badge status="processing" text="Step 3" />
        <a-card  style="overflow-x: auto; max-width: 100%;">
          <a-form>
            <div v-for="(matrix, key) in weightsMap" :key="key">
              <a-tag color="orange">
                {{ key }}:
              </a-tag>
              <table style="margin-top: 5px">
                <tr v-for="(row, rowIndex) in matrix" :key="`row-${rowIndex}`">
                  <td v-for="(value, colIndex) in row" :key="`col-${colIndex}`">
                    <a-input-number v-model="weightsMap[key][rowIndex][colIndex]" @change="value => updateWeightsMap(key, rowIndex, colIndex, value)" />
                  </td>
                </tr>
              </table>
              <br>
            </div>
          </a-form>

          <a-divider />

          <div v-for="(array, key) in biasMap" :key="key">
            <a-tag color="blue">
              {{ key }}:
            </a-tag>
            <table style="margin-top: 5px">
              <tr>
                <td v-for="(value, index) in array" :key="`bias-${index}`">
                  <a-input-number type="number" v-model="biasMap[key][index]" @change="value => updateBiasMap(key, index, value)"/>
                </td>
              </tr>
            </table>
            <br>
          </div>
        </a-card>
      </a-col>

      <a-col :span="7" :offset="1">
        <a-card>
          <a-form-item label="LOGS: ">
            <a-textarea :rows="31"  v-model:value="LOGS.join('\n')" />
          </a-form-item>
        </a-card>

      </a-col>


    </a-row>

  </div>
</template>

<script>


export default {
  name: "FeedForwardCalculator",

  data() {
    return {
      LOGS: [],
      form: this.$form.createForm(this, { name: 'coordinated' }),
      networkParams: {
        Di: 2,
        D: 2,
        Do: 2,
      },

      inputs: [],
      outputs: [],
      hiddenUnits: [],
      showMatrixInputs: false,

      weightsMap: {},
      biasMap: {},
      calculateReady: false
    };

  },

  created() {
    this.reset()
  },

  methods: {

    initializeNetwork() {

      let inputSize = this.networkParams.Di;
      let outputSize = this.networkParams.Do;
      let unitsPerLayer = this.hiddenUnits;

      // Initialize hidden layer weights and biases
      for (let i = 0; i < unitsPerLayer.length; i++) {
        let inputsLength = i === 0 ? inputSize : unitsPerLayer[i - 1];
        this.weightsMap['W' + (i)] = this.createMatrix(inputsLength, unitsPerLayer[i]);
        this.biasMap['B' + (i)] = Array(unitsPerLayer[i]).fill(0);
      }

      // Initialize output layer weights and biases
      this.weightsMap['W' + unitsPerLayer.length] = this.createMatrix(unitsPerLayer[unitsPerLayer.length - 1], outputSize);
      this.biasMap['B' + unitsPerLayer.length] = Array(outputSize).fill(0);

    },

    createMatrix(row, column) {
      return Array(row).fill(0).map(() => Array(column).fill(0));
    },

    calculateOutput() {


      this.LOGS = [];

      this.LOGS.push("Calculating Output: ");
      this.LOGS.push("---------");

      let currentOutput = this.inputs;

      const weightKeys = Object.keys(this.weightsMap);
      const biasKeys = Object.keys(this.biasMap);

      for (let i = 0; i < weightKeys.length; i++) {
        const weightKey = weightKeys[i];
        const biasKey = biasKeys[i];

        this.LOGS.push(`Inputs: ${JSON.stringify(currentOutput)}`);

        currentOutput = this.multiplyWeights(currentOutput, this.weightsMap[weightKey]);
        currentOutput = this.addBiases(currentOutput, this.biasMap[biasKey]);
        currentOutput = this.applyActivation(currentOutput);

        this.LOGS.push(`Output${i + 1}: ${JSON.stringify(currentOutput)}`);
        this.LOGS.push("---------");
      }

      this.LOGS.push(`Final Output: ${JSON.stringify(currentOutput)}`);
    },

    relu(x) {
      return Math.max(0, x);
    },

    multiplyWeights(inputs, weights) {
      console.log("Inputs: ", inputs)
      console.log("Weights: ", weights)

      let output = [];
      for (let i = 0; i < weights[0].length; i++) {
        let netInput = 0;
        for (let j = 0; j < inputs.length; j++) {
          netInput += inputs[j] * weights[j][i];
        }

        output.push(netInput);
      }

      output = this.roundNumbersInArray(output);
      this.LOGS.push("Multiplying Weights: ");
      this.LOGS.push(JSON.stringify(output))
      return output
    },

    addBiases(inputs, biases) {
      let output = inputs.map((input, index) => input + biases[index]);
      output = this.roundNumbersInArray(output);
      this.LOGS.push("Adding Biases: ");
      this.LOGS.push(JSON.stringify(output))
      return output;
    },


    applyActivation(inputs) {
      let output = inputs.map(input => this.relu(input));
      output = this.roundNumbersInArray(output);
      this.LOGS.push("Applying Activation: ");
      this.LOGS.push(JSON.stringify(output))
      return output;
    },

    reset() {
      this.weightsMap = {};
      this.biasMap = {};
      this.inputs = Array(this.networkParams.Di).fill(0);
      this.outputs = Array(this.networkParams.Do).fill(0);
      this.hiddenUnits = Array(this.networkParams.D).fill(0);
      this.showMatrixInputs = true;
    },

    updateWeightsMap(key, rowIndex, colIndex, value) {
      this.weightsMap[key] = [...this.weightsMap[key]];
      this.$forceUpdate();
    },

    updateBiasMap(key, index, value) {
      this.biasMap[key] = [...this.biasMap[key]];
      this.$forceUpdate();
    },

    roundNumbersInArray(arr, precision = 2) {
      return arr.map(num => Number(num.toFixed(precision)));
    }

  },

  computed: {
  },

  watch: {
  }
}
</script>

<style scoped>
</style>