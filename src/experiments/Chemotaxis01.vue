<template>
  <div class="home">
    <dat-gui
      closeText="Close controls"
      openText="Open controls"
      closePosition="bottom">
      <dat-number v-model="x_good" show-slider :min="0" :max="5" :step="0.001" label="x_good"/>
      <dat-number v-model="x_var_log" show-slider :min="-5" :max="1" :step="1" label="x_var (log)"/>
    </dat-gui>

    <button @click="step">Step</button>
    <button @click="run">Run</button>
    <button @click="reset">Reset</button>
    <button @click="draw">Draw</button>
    <div id="draw-shapes"></div>
    <pre class="output">
solution: {{Math.pow((9/2),(1/3))}}
x_var: {{x_var}}
x_good: {{x_good}}
cost_good: {{cost(x_good)}}
x_log: {{x_log}}</pre>
  </div>
</template>

<script>
import Vue from 'vue'
import DatGui from '@cyrilf/vue-dat-gui'
import Two from 'two.js'

Vue.use(DatGui)

// Standard Normal variate using Box-Muller transform.
function randn() {
  var u = 0, v = 0;
  while(u === 0) u = Math.random(); //Converting [0,1) to (0,1)
  while(v === 0) v = Math.random();
  return Math.sqrt( -2.0 * Math.log( u ) ) * Math.cos( 2.0 * Math.PI * v );
}

function rand() {
  return (Math.random() - 0.5);
}

export default {
  data() {
    return {
      x_good: 3,
      r_good: 1e99,
      x_var: 0.1,
      x_log: [],
    }
  },
  mounted() {
    // Make an instance of two and place it on the page.
    let elem = document.getElementById('draw-shapes');
    let params = { width: 600, height: 100 };
    let two = new Two(params).appendTo(elem);

    two.scene._matrix.manual = true;
    two.scene._matrix.translate(0, two.height).scale(1, -1);

    this.two = two;
    window.mytwo = two;

    this.draw();
  },
  computed: {
    x_var_log: {
      get() {
        return Math.log10( this.x_var )
      },
      set(v) {
        this.x_var = Math.pow( 10, v )
      },
    },
  },
  methods: {
    reset() {
      this.x_good = 3;
      this.x_log = [];
      this.draw();
    },
    run() {
      console.log("run")
      this.x_good = this.optimize( this.x_good, this.x_var );
      this.draw();
    },
    cost(x) {
      return ( 20 * x * x ) + 180 * (1/x);
    },
    optimize( x_good, x_var = 0.1 ) {
      let r_good = 1e99;
      for ( let n = 0; n < 50; n++ ) {
        let x = x_good + x_var * randn();
        let r = this.cost(x);
        if ( r < r_good ) {
          x_good = x;
          r_good = r;
          this.x_log.push(x_good);
        }
      }
      return x_good;
    },
    step() {
      let x = this.x_good + this.x_var * randn();
      let r = this.cost(x);
      if ( r < this.r_good ) {
        this.x_good = x;
        this.r_good = r;
        this.x_log.push( x );
      }
    },
    draw() {
      const two = this.two;

      two.clear();

      // two has convenience methods to create shapes.
      let list = this.x_log.map((y,x) => {
        let circle = two.makeCircle(x * 5, y * 15, 2);

        // The object returned has many stylable properties:
        circle.fill = '#FF8000';
        circle.linewidth = 0;

        return circle
      });

      // let line = two.makeLine(0, 0, 600, 100)
      let line = two.makeLine(0, this.x_good * 15, 600, this.x_good * 15)

      // Groups can take an array of shapes and/or groups.
      var group = two.makeGroup( ...list );

      // And have translation, rotation, scale like all shapes.
      // group.translation.set(two.width / 2, two.height / 2);
      // group.translation.set(two.width, 0);
      // group.rotation = Math.PI;
      // group.scale = 2;

      // var rect = two.makeRectangle(213, 100, 100, 100);
      // rect.fill = 'rgb(0, 200, 255)';
      // rect.opacity = 0.75;
      // rect.noStroke();

      // Don't forget to tell two to render everything
      // to the screen
      two.update();
    }
  },
}
</script>

<style>
.vue-dat-gui .control-item.number .control .slider {
  flex: 0 0 100px;
  width: 100px;
}
.output {
  border: 2px solid black;
  text-align: left;
  padding: 1em;
  margin: 1em 6em;
}
#draw-shapes {
  border: 2px solid black;
  text-align: left;
  margin: 1em auto;
  width: 600px;
}
</style>
