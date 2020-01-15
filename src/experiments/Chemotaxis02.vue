<template>
  <div class="home">
    <dat-gui
      v-if="false"
      closeText="Close controls"
      openText="Open controls"
      closePosition="bottom">
      <dat-number v-model="h_good" show-slider :min="0" :max="500" :step="1" label="h_good"/>
      <dat-number v-model="r_good" show-slider :min="0" :max="1" :step="0.001" label="r_good"/>
      <dat-boolean v-model="show_only_good" label="show_only_good"/>
    </dat-gui>

    <button @click="step(); draw();">Step</button>
    <button @click="run(10)">Run 10 steps</button>
    <button @click="run(100)">Run 100 steps</button>
    <button @click="run(null)">Run inf</button>
    <button @click="stop()">Stop</button>
    <button @click="reset()">Reset</button>
    |
    <button @click="incvar()">+ Var</button>
    <button @click="decvar()">- Var</button>
    <!-- <button @click="draw()">Draw</button> -->

    <div class="boxes">
      <pre class="infobox">#steps:
{{steps}}</pre>
      <pre class="infobox">
Variance
Var. h: ± {{h_var}}
Var. r: ± {{r_var}}</pre>
      <pre class="infobox">
Best inputs:
h_good: {{h_good|round}} m
r_good: {{r_good|round}}</pre>
      <pre class="infobox">
Best cost:
c_good:
{{-objective(h_good, r_good)|round}} Mio $</pre>
    </div>
    <div id="draw-shapes-2"></div>
    <div id="draw-shapes-3"></div>

<pre v-if="!isProd" class="output">
best win: {{win_good|round}} Mio $
#steps: {{steps}} ({{log.length}})
</pre>

<pre v-if="!isProd" class="output">
cost ( h, r ) {
  // Revenue from appartments
  let Em = 1 * ( 1 - r ) * ( 0.5 * h * Math.sqrt( h ) );

  // Revenue from shops
  let Es = Math.sqrt( Em ) * 100 * r;

  // Land
  let Kl = 500;

  // Construction
  let Kc = 250 + (700/(200*200)) * h * h;

  // Air rights
  let Ka = (200/220) * h;

  let K = Kc + Ka + Kl;
  let E = Em + Es;

  return E - K
}
  </pre>
    <!-- <pre class="output">log: {{log}}</pre> -->
  </div>
</template>

<script>
import Vue from 'vue'
import Two from 'two.js'

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

function limit(v, l, u) {
  return Math.min( Math.max(l, v), u );
}

function initState() {
  return {
    isProd: process.env.NODE_ENV !== 'development',
    h_good: 100, // 300
    r_good: 0.0, // 0.05
    h_var: 2,
    r_var: 0.02,
    win_good: -1e99,
    log: [],
    steps: 0,
    show_only_good: false,
  }
}

export default {
  data() {
    return initState();
  },
  watch: {
    show_only_good() { this.draw() },
  },
  mounted() {
    // Make an instance of two and place it on the page.
    let twoObj = {};
    for ( let domNode of ['draw-shapes-2', 'draw-shapes-3']) {
      let elem = document.getElementById(domNode);
      let params = { width: 600, height: 200, type: Two.Types.canvas };
      let two = new Two(params).appendTo(elem);
      two.scene._matrix.manual = true;
      two.scene._matrix.translate(0, two.height).scale(1, -1);
      twoObj[ domNode ] = two;
    }

    this.two = twoObj['draw-shapes-2']
    this.two2 = twoObj['draw-shapes-3']
    window.two = this.two;
    window.Two = Two;

    this.reset();
  },
  destroyed() {
    window.cancelAnimationFrame(this.animationId);
    this.animationId = undefined;
  },
  methods: {
    reset() {
      this.stop();
      Object.assign( this, initState() );
      this.drawInit( this.two2, false );
      this.drawInit( this.two, true );
      this.h_prev = null;
      this.r_prev = null;
    },
    incvar() {
      this.h_var *= 2;
      this.r_var *= 2;
    },
    decvar() {
      this.h_var /= 2;
      this.r_var /= 2;
    },
    stop() {
      clearInterval( this.interval );
      this.draw();
    },
    run(steps = 10) {
      this.stop();
      let n = 1;
      this.interval = setInterval(() => {
        this.step();
        this.animationId = window.requestAnimationFrame(this.draw);
        n += 1;
        if ( steps && n > steps ) {
          this.stop();
        }
      }, 15);
    },
    objective(h, r) {

      // r = Math.min( Math.max(0, r), 1 );
      let Em = 1 * ( 1 - r ) * ( 0.5 * h * Math.sqrt( h ) ); // Revenue from appartments
      let Es = Math.sqrt( Em ) * 100 * r; // Revenue from shops

      let Kl = 500;                         // Land
      let Kc = 250 + (700/(200*200)) * h * h; // Construction
      let Ka = (200/220) * h;               // Air rights

      let K = Kc + Ka + Kl;
      let E = Em + Es;
      // console.log("cost", { Em, Es, Kl, Kc, Ka, K, E, r, h })
      return ( E - K );
    },
    step() {
      this.steps += 1;
      let h = this.h_good + this.h_var * randn();
      let r = this.r_good + this.r_var * randn();
      r = limit(r, 0, 1);
      let win = this.objective(h, r);
      if ( win > this.win_good ) {
        this.h_good = h;
        this.r_good = r;
        this.win_good = win;
        this.log.push([this.steps, h, r, win, true]);
      } else {
        this.log.push([this.steps, h, r, win, false]);
      }
    },
    drawInit( two, withText = true ) {
      two.clear();

      var styles = {
        alignment: "right",
        size: 10,
        family: "sans-serif",
      };

      let g = two.makeGroup()

      for ( let n = 0; n <= 20; n++ ) {
        let n_y = (n/20)*600;
        let line = two.makeLine(n_y, 0, n_y, 200);
        line.stroke = "#aaa";
        if ( withText ) {
          let text = two.makeText(n_y, n_y - 5, 10, styles)
          g.add(text)
        }
      }

      for ( let n = 0; n <= 10; n++ ) {
        let n_x = (n/10)*200;
        let line = two.makeLine(0, n_x, 600, n_x);
        line.stroke = "#aaa";
        if ( withText ) {
          let text = two.makeText(Math.round(10 - n_x / 20) / 10, 20, 10+n_x, styles)
          g.add(text)
        }
      }

      g._matrix.manual = true;
      g._matrix.translate(0, this.two.height).scale(1, -1);

      if ( !this.skyscraper ) {
        this.skyscraper = two.makeRectangle(550, 25 + 1, 40, 50);
        this.skyscraper.linewidth = 1;
        this.skyscraper.stroke = 'black';
        this.skyscraper.fill = 'none';
        this.skyscraperRatio = two.makeRectangle(550, 1, 40, 0);
        this.skyscraperRatio.fill = 'grey'
      }

      two.update()
    },
    draw() {
      const two = this.two;
      const two2 = this.two2;

      if ( this.skyscraper ) {
        let h_sketch = this.h_good / 2;
        this.skyscraper.translation.y = 1 + h_sketch / 2;
        this.skyscraper.height = h_sketch
        this.skyscraperRatio.translation.y = 1 + h_sketch * (this.r_good) / 2;
        this.skyscraperRatio.height = h_sketch * (this.r_good);
      }

      // two has convenience methods to create shapes.
      this.log.filter(data => !data[5]).forEach((data) => {
        let x = data[0]
        let h = data[1];
        let r = data[2];
        let win = data[3];
        let is_better = data[4];

        if ( this.show_only_good && !is_better ) {
          return null;
        }

        let circle = two.makeCircle( h, r * 200, 2);
        circle.fill = is_better ? '#FF8000' : '#8000FF';
        circle.linewidth = 0;
        circle.stroke = 'none';

        let circleWin = two2.makeCircle( x, 50 + win/5, 2);
        circleWin.fill = is_better ? '#FF8000' : '#8000FF';
        circleWin.linewidth = 0;
        circleWin.stroke = 'none';

        if ( this.h_prev && this.r_prev ) {
          two.makeLine( this.h_prev, this.r_prev * 200, h ,r * 200 );
        }

        if ( is_better ) {
          this.h_prev = h;
          this.r_prev = r;
        }

        data[5] = true;
      });

      two.update();
      two2.update();

      // The object returned has many stylable properties:
      // let circle1 = two.makeCircle(x * 5, y1 * 15, 2);
      // let circle2 = two.makeCircle(x * 5, y2 * 15, 2);
      // circle2.fill = '#80FF00';
      // circle2.linewidth = 0;
      // Groups can take an array of shapes and/or groups.
      // var group = two.makeGroup( ...list );
    }
  },
  filters: {
    round( value ) {
      return Math.round( value * 100.0 ) / 100.0;
    }
  }
}
</script>

<style>
.output {
  border: 2px solid black;
  text-align: left;
  padding: 1em;
  margin: 1em 6em;
}
.boxes {
  display: flex;
  flex-wrap: wrap;
  text-align: left;
  margin: 1em auto;
  width: 600px;
}
.boxes > * {
  width: 21%;
}
.infobox {
  border: 2px solid black;
  padding: 0.5em;
  margin-right: 2px;
}
#draw-shapes-2,
#draw-shapes-3 {
  /* border: 2px solid black; */
  text-align: left;
  margin: 1em auto;
  width: 600px;
}
</style>
