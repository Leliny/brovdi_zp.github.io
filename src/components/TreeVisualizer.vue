<template>
  <div
    id="tree-visualizer-handler"
    class="flex tree-container"
  >

    <div class="flex buttons ">
      <InsertInput
        :disabled="isAnimating"
        @myEvent="insertInputEvent"
      />
      <DeleteInput
        :disabled="isAnimating"
        :onlypop="onlypop"
        @myEvent="deleteInputEvent"
      />
    </div>
    <Visualizer
      :sequences="sequences"
      :current-sequence-number="currentSequenceNumber"
      :current-frame-number="currentFrame"
      :tree-type="$route.params.code"
    />
    <div class="arrow-button-container flex mx-auto my-2">
      <div class="history-btn-container">
        <HistoryButtons
          id="history.buttons"
          class="mx-4"
          :current="currentSequenceNumber"
          :length="sequences.length"
          :disabled="historyButtonsAreDisabled"
          label="History"
          @HistoryEvents="changeCurrentSequence"
        />
      </div>
      <v-btn
        class="button"
        color="primary"
        :disabled="isAnimating"
        @click="replay"
      >
        Replay <v-icon>mdi-play</v-icon>
      </v-btn>
      <div class="step-btn-container">
        <HistoryButtons
          id="step-buttons"
          class="mx-4"
          :current-frame="currentFrame"
          :length="currentSequence ? currentSequence.frames.length : 0"
          :disabled="stepButtonsAreDisabled"
          label="Animation"
          @HistoryEvents="changeCurrentFrame"
        />
      </div>
    </div>
  </div>
</template>

<script>
import { createRouter, createWebHistory } from 'vue-router';
import InsertInput from './shared/InsertInput.vue';
import DeleteInput from './shared/DeleteInput.vue';
import Visualizer from './shared/Visualizer.vue';
import HistoryButtons from './shared/HistoryButtons.vue';
import Sequence from '../assets/visualizer/frame';
import BTree, { BTreeNode } from '../assets/implementations/btree';
import MainMenu from "@/components/MainMenu.vue";

// Створення екземпляра маршрутизатора
const router = createRouter({
  history: createWebHistory(),
  routes: [
    { path: '/', component: MainMenu },
    { path: '/:pathMatch(.*)*', component: MainMenu },
  ]
});

export default {
  name: 'TreeVisualizerHandler',
  components: {
    InsertInput,
    DeleteInput,
    Visualizer,
    HistoryButtons,
  },
  data() {
    return {
      /** @type {BTree | MinHeap} */
      tree: null,
      /** @type {Sequence[]} */
      sequencesList: [],
      insertionQueue: [],
      currentSequenceNumber: 0,
      currentFrame: 0,
      onlypop: true,
      isAnimating: false,
      title: '',
    };
  },
  computed: {
    sequences() {
      return this.sequencesList;
    },
    currentSequence() {
      return this.sequencesList[this.currentSequenceNumber];
    },
    stepButtonsAreDisabled() {
      if (!this.currentSequence) {
        return {
          backButtonIsDisabled: true, forwardButtonIsDisabled: true,
        };
      }
      const backButtonIsDisabled = this.currentFrame <= 0;
      const forwardButtonIsDisabled = this.currentFrame >= this.currentSequence.frames.length - 1;
      return { backButtonIsDisabled, forwardButtonIsDisabled, isAnimating: this.isAnimating };
    },
    historyButtonsAreDisabled() {
      if (!this.sequencesList) {
        return { backButtonIsDisabled: true, forwardButtonIsDisabled: true };
      }
      const backButtonIsDisabled = this.currentSequenceNumber <= 0;
      const forwardButtonIsDisabled = this.currentSequenceNumber >= this.sequencesList.length - 1;
      return { backButtonIsDisabled, forwardButtonIsDisabled, isAnimating: this.isAnimating };
    },
  },
  mounted() {
    switch (this.$route.params.code) {
      case 'b-tree':
        this.tree = new BTree(2);
        this.onlypop = false;
        this.title = 'B Tree';
        this.tree.root = new BTreeNode(true);
        this.tree.root.tree = this.tree;
        return;
      default:
        this.$router.push('/');
    }
  },
  methods: {
    goBack() {
      router.back();
    },
    insertInputEvent(event) {
      event.split(',').forEach((value) => {
        if (value) {
          this.insertionQueue.push(value);
        }
      });
      this.dequeueInsert();
    },
    dequeueInsert() {
      const newSequence = new Sequence();
      this.sequencesList.push(newSequence);
      this.currentSequenceNumber = this.sequencesList.length - 1;
      const sequence = this.tree.insert(this.insertionQueue[0]);
      this.insertionQueue.splice(0, 1);
      this.addSequenceAsync(sequence);
    },
    deleteInputEvent(event) {
      const newSequence = new Sequence();
      this.sequencesList.push(newSequence);
      this.currentSequenceNumber = this.sequencesList.length - 1;
      this.addSequenceAsync(this.tree.delete(event));
    },
    replay() {
      if (this.currentSequence && this.currentSequence.frames) {
        const frames = [];
        this.currentSequence.frames.forEach((f) => frames.push(f));
        this.currentSequence.frames = [];
        this.addSequenceAsync({ frames });
      }
    },
    addSequenceAsync(sequence) {
      this.isAnimating = true;
      this.currentSequence.addFrame(sequence.frames[0]);
      this.currentFrame = 0;
      for (let i = 1; i < sequence.frames.length; i += 1) {
        setTimeout(() => {
          this.currentSequence.addFrame(sequence.frames[i]);
          this.currentFrame += 1;
          this.sequencesList = Object.assign(this.sequencesList);
          if (i === sequence.frames.length - 1) {
            this.isAnimating = false;
            if (this.insertionQueue.length > 0) {
              this.dequeueInsert();
            }
          }
        }, i * 500);
      }
    },
    changeCurrentFrame(event) {
      this.currentFrame += event;
      if (this.currentFrame < 0) {
        this.currentFrame = 0;
      }
      if (this.currentFrame > this.sequencesList[this.currentSequenceNumber].frames.length - 1) {
        this.currentFrame = this.sequencesList[this.currentSequenceNumber].frames.length - 1;
      }
    },
    changeCurrentSequence(event) {
      this.currentSequenceNumber += event;
      if (this.currentSequenceNumber < 0) {
        this.currentSequenceNumber = 0;
      }
      if (this.currentSequenceNumber > this.sequencesList.length - 1) {
        this.currentSequenceNumber = this.sequencesList.length - 1;
      }
      this.currentFrame = this.currentSequence.frames.length - 1;
    },
  },
};
</script>


<style scoped>
  .structies-button:hover {
    background-color:#3c48fa;
  }
  .back {
    position: absolute;
    left: 0;
    top: 0;
    background-color:#edf2f7;
  }
  .buttons {
    place-content: space-around;

    flex: 0 0 auto;
  }
  .tree-container {
    width: 100%;
    height: 87vh;
    flex-direction: column;
    font-family: 'Inconsolata', monospace;
  }
</style>
