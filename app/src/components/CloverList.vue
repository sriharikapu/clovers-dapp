<template>
  <div class="">
    <div class="bg-green  white pb2 md-p3 flex justify-around">
      <div class="center" v-for="sym in symTypes" >
        <div class='px1'>{{symmetries[sym]}} x</div>
        <div :class="sym" class="symmetry-type"></div>
        <div>{{symValues[sym]}} ♧</div>
      </div>
    </div>

    <div id="break-up" class="bg-gray white p2 md-p3 flex justify-around shadow-bottom">
      <div>{{ allClovers.length }} Clovers Claimed</div>
      <div>{{ symmetries.RotSym }} Rotational</div>
      <div>{{ symmetries.X0Sym + symmetries.Y0Sym }} Perpindicular</div>
      <div>{{ symmetries.XYSym + symmetries.XnYSym }} Diagonal</div>
    </div>

    <div v-if="allClovers.length" class="mt2 px2">



      <div class="hide">
        <form
          class="border-bottom inline-block my1"
          @submit.prevent="search">
          <input class="input" v-model="search" placeholder="search">
        </form>
      </div>


       <div class=" max-width-4 mx-auto flex justify-between align-middle py3" >
          <div class=" " :class="{hide: pagedTotal === 1}">
            <button
            class="btn btn-outline mb1 blue"
            :disabled="!prevPossible"
            @click="chPage(-1)">←</button>&nbsp;
          </div>

        <div class="">
          <span
          @click="clickSort(i)"
          :class="sortableClass(i)"
          class="inline-block mx2 pointer no-select"
          v-html="sort"
          v-for="sort, i in sortable"></span>
        </div>

          <div class=" " :class="{hide: pagedTotal === 1}">
            <button
            class="btn btn-outline mb1 blue"
            :disabled="!nextPossible"
            @click="chPage(1)">→</button>&nbsp;
          </div>
        </div>

      <ul ref="cloverList" class="list-reset max-width-5 mx-auto flex flex-wrap mxn2 center justify-center">
        <li v-for="board in cloversSliced" :key="board.board" class="px2 mb3">
          <clover-grid-item :by-flip="sortableIndex == 0 || sortableIndex == 3" :key="board.board" :board="board"></clover-grid-item>
        </li>
        <li class="list-reset flex flex-wrap mxn2 center justify-center" v-if="cloversSliced.length === 0">Go claim some Clovers...

        </li>
      </ul>
      <div class="hide">
        <span
        class="btn btn-outline mb1 orange"
        @click="limit = amount"
        v-for="amount in limits"
        :class="{'bg-red': limit === amount}"
        v-html="amount"></span>
      </div>





         <div class=" max-width-4 mx-auto flex justify-between align-middle" >
          <div class=" " :class="{hide: pagedTotal === 1}">
            <button
            class="btn btn-outline mb1 blue"
            :disabled="!prevPossible"
            @click="chPage(-1)">←</button>&nbsp;
          </div>

          <div class="center mb2"  :class="{hide: pagedTotal === 0}">
            <span>{{paged}} / {{pagedTotal}}</span>
          </div>

          <div class=" " :class="{hide: pagedTotal === 1}">
            <button
            class="btn btn-outline mb1 blue"
            :disabled="!nextPossible"
            @click="chPage(1)">→</button>&nbsp;
          </div>
        </div>

    </div>
  </div>
</template>

<script>
  import { mapGetters } from 'vuex'
  export default {
    name: 'CloverList',
    data () {
      return {
        paged: 1,
        limit: 18,
        limits: [5, 10, 20, 50, 100],
        asc: true,
        sortableIndex: 3,
        symTypes: ['RotSym', 'Y0Sym', 'X0Sym', 'XYSym', 'XnYSym'],
        sortable: ['Date Flipped', 'Date Found', 'Current Price', 'Times Flipped'],
        search: null
      }
    },
    watch: {
      limit () {
        this.paged = 1
      }
    },
    props: {
      filter: {
        type: String,
        default: null
      }
    },
    computed: {
      symValues () {
        return {
          RotSym: this.clover.calcFindersFees(this.symmetries, true),
          Y0Sym: this.clover.calcFindersFees(this.symmetries, false, true),
          X0Sym: this.clover.calcFindersFees(this.symmetries, false, false, true),
          XYSym: this.clover.calcFindersFees(this.symmetries, false, false, false, true),
          XnYSym: this.clover.calcFindersFees(this.symmetries, false, false, false, false, true)
        }
      },
      pagedTotal () {
        return Math.floor(this.cloversSorted.length / this.limit) + (this.cloversSorted.length % this.limit && 1)
      },
      prevPossible () {
        return this.paged > 1
      },
      nextPossible () {
        return this.paged < this.pagedTotal
      },
      startSlice () {
        return this.limit * (this.paged - 1)
      },
      endSlice () {
        return this.limit * this.paged
      },
      cloversSliced () {
        return this.cloversSorted.slice(this.startSlice, this.endSlice)
      },
      cloversSorted () {
        return this.allClovers.sort((a, b) => {
          switch (this.sortableIndex) {
            case (0):
              return this.asc ? b.modified - a.modified : a.modified - b.modified
            case (1):
              return this.asc ? b.created - a.created : a.created - b.created
            case (2):
              return this.asc ? this.currPrice(b) - this.currPrice(a) : this.currPrice(a) - this.currPrice(b)
            case (3):
              return this.asc ? b.previousOwners.length - a.previousOwners.length : a.previousOwners.length - b.previousOwners.length
          }
        }).filter((c) => {
          if (!this.search && !this.filter) return c
          return c.previousOwners.slice(-1).filter((p) => {
            return (p.name && p.name.search(this.search || this.filter) > -1) || p.address.search(this.search || this.filter) > -1
          }).length || // owner
          c.previousOwners.slice(0, 1).filter((p) => {
            return (p.name && p.name.search(this.search || this.filter) > -1) || p.address.search(this.search || this.filter) > -1
          }).length || // founder
          c.name && c.name.search(this.search || this.filter) > -1 || // board name
          c.first32Moves.search(this.search || this.filter) > -1 || // moves
          c.first32Moves.search(this.search || this.filter) > -1 || // moves
          c.lastMoves.search(this.search || this.filter) > -1 || // moves
          c.board.search(this.search || this.filter) > -1 // board
        })
      },
      ...mapGetters([
        'clover',
        'allClovers',
        'usernames',
        'clovernames',
        'symmetries'
      ])
    },
    methods: {
      chPage (amount) {
        window.scrollTo(0, this.$refs.cloverList.offsetTop - 150)
        this.$nextTick(() => {
          this.paged += amount
        })
      },
      currPrice (c) {
        return c.previousOwners.length === 1 ? c.lastPaidAmount : (c.lastPaidAmount * 2)
      },
      sortableClass (i) {
        if (i !== this.sortableIndex) return 'muted'
        return {
          gray: true,
          asc: this.asc,
          desc: !this.asc
        }
      },
      clickSort (i) {
        if (i !== this.sortableIndex) {
          this.sortableIndex = i
        } else {
          this.asc = !this.asc
        }
      }
    }
  }
</script>

<style lang="scss">
.shadow-bottom {
  box-shadow: 0px 4px 7px 1px rgba(0, 0, 0, .2);
}
  @media only screen and (max-width: 768px) {
  #break-up {
    flex-wrap:wrap;
    > * {
      text-align: center;
      width:100%;
    }
  }
}
</style>
