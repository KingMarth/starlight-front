<template>
  <div>
    <div class="form-group mb-0">
      <input
        type="text"
        class="search form-control"
        placeholder="Search for a name here"
        v-model="search"
      >
    </div>
    <ul class="list-group">
      <li class="list-group-item list-group-item-primary">
        Survivors and structures
      </li>
      <SurvivorComponent
        v-for="survivor in displayedSurvivors"
        v-bind:survivor="survivor"
        :key="survivor.id"
      ></SurvivorComponent>
      <li class="list-group-item list-group-item-violet">
        Magics
      </li>
      <MagicComponent
        v-for="magic in displayedMagics"
        v-bind:magic="magic"
        :key="magic.id"
      ></MagicComponent>
    </ul>
  </div>
</template>
<script lang="ts">
import {
  Component,
  Prop,
  Vue,
  Watch,
} from 'vue-property-decorator';
import uuid from 'uuid';
import Survivor from '../Survivor';
import SurvivorComponent from './Survivor.vue';
import MagicComponent from './Magic.vue';
import Requests from '../Requests';
import Magic from '@/Magic';

/**
 * Return a filtering function based on the string given
 * keyword supported:
 *  `@pos:x/y` : Search for all survivors in the hexagon x,y
 *  `@job:...,...`: Search for all survivors with the given jobs
 *  `@items: ...,...`: Search for all survivors with the given items
 */
function funct(str: string): (surv: Survivor) => boolean {
  const match = str.match(/@(.*):(.*)/);
  if (match != null) {
    const [, fun, value] = match;

    switch (fun) {
      case 'pos':
        return (surv: Survivor) => value.split(',').map((s) => {
          const [x, y] = s.split('/');
          return { x: Number(x), y: Number(y) };
        }).find(pos => pos.x === surv.Position.x && pos.y === surv.Position.y) != null;
      case 'jobs':
        return (surv: Survivor) => value.split(',')
          .every(job => surv.Jobs.reduce((acc, j) => acc + j.Name, '').toLowerCase().match(job.toLowerCase()) != null);
      case 'items':
        return (surv: Survivor) => value.split(',')
          .every(item => surv.Items.map(ite => ite.toLowerCase().replace(' ', '-')).find(ite => ite.match(item.toLowerCase()) != null) != null);
      default:
        return () => true;
    }
  } else {
    return (surv: Survivor) => surv.Name.toLowerCase().match(str.toLowerCase()) != null;
  }
}

@Component({
  components: {
    SurvivorComponent,
    MagicComponent,
  },
})
export default class SideBar extends Vue {
  @Prop() private msg!: string;

  private survivors: Survivor[] = [];

  private magics: Magic[] = [];

  private displayedSurvivors: Survivor[] = [];

  private displayedMagics: Magic[] = [];

  private search: string = '';

  mounted() {
    fetch(Requests.AllInfoRequest(Requests.BASE_URL)).then(d => d.json())
      .then((infos) => {
        const survs = infos.Survivors;
        if (!(survs instanceof Array)) throw new Error('Data is corrupted');

        this.survivors = survs.map((inhab) => {
          const numItems = inhab.items.toString() === '' ? 0 : inhab.items.length;

          const survivor: Survivor = {
            Name: inhab.Name,
            Description: inhab.Description,
            Position: inhab.Position,
            Health: { current: inhab.Health, max: inhab.MaxHealth },
            Items: inhab.items,
            Conditions: null,
            Jobs: inhab.jobs,
            id: uuid.v4(),
          };

          // Pretifiying the conditions and jobs
          if (!inhab.conditions.includes('')) {
            survivor.Conditions = inhab.conditions.toString().replace(/,/g, ', ');
          }

          return survivor;
        });

        const magics = infos.Magic;
        if (!(magics instanceof Array)) throw new Error('Data is corrupted');

        this.magics = magics.map((magic) => {
          const mag: Magic = {
            id: uuid.v4(),
            Name: magic.Name,
            Constelation: magic.Constelation,
            StarChart: magic.StarChart,
            Spell: magic.Spell,
            Levels: magic.Levels,
            OmCombo: magic.OmCombo,
          };

          return mag;
        });

        this.displayedMagics = this.magics;
        this.displayedSurvivors = this.survivors;
      }).catch(err => console.error(err));
    // Listen for goto
    const beacon = document.querySelector<HTMLElement>('#beacon');
    if (beacon instanceof HTMLElement) {
      beacon.addEventListener('selected', (e: Event) => {
        if (e instanceof CustomEvent && e.detail instanceof Array && e.detail.length !== 0) {
          this.search = `@pos:${e.detail.map((p: {x: number, y: number}) => `${p.x}/${p.y}`).toString()}`;
        } else {
          this.search = '';
        }
      });
    }
  }

  @Watch('search')
  onSearchChange(val: string, oldVal: string) {
    const querys = val.split(' ');
    this.displayedSurvivors = querys
      .map(funct)
      .reduce((acc: Survivor[], filter: (s: Survivor) => boolean) => acc.filter(filter),
        this.survivors);
  }
}
</script>

<style lang="scss" scoped>
ul {
  height: calc(100% - 6rem);
  overflow-x: hidden;
  overflow-y: auto;
}
.list-group-item-violet {
  color: white;
  background-color: #8c408c;
}
</style>
