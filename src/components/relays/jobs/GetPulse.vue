<template>
  <span 
    v-if="this.store.jobs.getActiveSlug === slug"
    class="text-inherit">
    <span class="text-inherit">
      <svg class="animate-spin mr-1 -mt-0.5 h-4 w-5 text-white inline" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
        <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
        <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
      </svg>
      {{ isSingle ? `taking ${relayFromUrl}'s pulse` : "taking nostr's pulse" }}
    </span>
  </span>
</template>

<style scoped>

</style>

<script>
import { defineComponent } from 'vue'

import crypto from 'crypto'
import decodeJson from 'unescape-json'

import { setupStore } from '@/store'

import RelayMethods from '@/shared/relays-lib.js'
import SharedComputed from '@/shared/computed.js'

import { relays } from '../../../../relays.yaml'
import { RelayPool } from 'nostr'

const localMethods = {
  invalidatePulse(){
    // if( (!this.isExpired(this.slug, 1) && !force) && !this.isSingle ) 
    //   return
    
    
      this.queueJob(
        this.slug, 
        () => this.jobPulses(),
        true
      )
  },
  async jobPulses(){
    if(!this.store.status.historyNode)
      return this.store.jobs.completeJob(this.slug)
    const subid = crypto.randomBytes(20).toString('hex')
    const pulsesByEvent = new Object()
    let total = 48,
        count = 0
    await new Promise( resolve => {
      const pool = new RelayPool( ['wss://history.nostr.watch'], { reconnect: false } )
      const uniques = new Set()
      let timeout = setTimeout( () => { 
        resolve()
        this.closePool(pool)
      }, 20000 )
      pool
        .subscribe(subid, {
          kinds:    [1411],
          limit:    total, //12 hours 
          authors:  ['b3b0d247f66bf40c4c9f4ce721abfe1fd3b7529fbc1ea5e64d5f0f8df3a4b6e6'],
          '#e':     [this.store.prefs.region],
        })
      
      pool
        .on('event', (relay, sub_id, event) => {
          if(sub_id !== subid)
            return
          
          if(uniques.has(event.created_at))
            return 

          // console.log(decodeJson(event.content).online?.length)
          
          uniques.add(event.created_at)

          pulsesByEvent[event.created_at] = decodeJson(event.content).online

          count++

          if(count !== total)
            return 
          
          resolve()
          this.closePool(pool)
        })
        .on('eose', () => {
          resolve()
          this.closePool(pool)
          clearTimeout(timeout)
        })
      
    })
    
    this.parsePulses(pulsesByEvent)
  },
  parsePulses(data){
    const allTimestamps = Object.keys(data),
          pulsesByRelayObj = new Object()

    allTimestamps.forEach( timestamp => {
      data[timestamp].forEach( relayData => {
        const relay = relayData[0],
              connectLatency = relayData[1]?.[0],
              readLatency = relayData[2]?.[0],
              writeLatency = relayData[3]?.[0]

        if( !(pulsesByRelayObj[relay] instanceof Object) )
          pulsesByRelayObj[relay] = allTimestamps.reduce( (acc, _timestamp) => {
            acc[_timestamp] = false
            return acc
          }, new Object())
        pulsesByRelayObj[relay][timestamp] = {
          connect: connectLatency,
          read: readLatency,
          write: writeLatency,
        }
      })
    })

    const allRelaysInPulses = Object.keys(pulsesByRelayObj)

    const pulses = new Object()

    allRelaysInPulses.forEach( relay => {
      pulses[relay] = new Array()
      //console.log(relay, pulsesByRelayObj[relay])
      Object.keys(pulsesByRelayObj[relay]).forEach( (timestamp_) => {
        pulses[relay].push(Object.assign(pulsesByRelayObj[relay][timestamp_], {
          date: timestamp_,
        }))
      })
      pulses[relay].sort( (h1, h2) => h1.date - h2.date )
      this.store.stats.addPulse(relay, pulses[relay])
      this.setUptimePercentage(relay)
      if(this.isSingle && this.relayFromUrl === relay)
        console.log('viola!', relay, this.getAbilityRate('connect', relay), this.getAbilityRate('read', relay), this.getAbilityRate('write', relay), pulses[relay])
    })
    
    this.store.jobs.completeJob(this.slug)
  },
  timeUntilRefresh(){
    return this.timeSince(Date.now()-(this.store.jobs.getLastUpdate(this.slug)+this.store.prefs.duration-Date.now())) 
  },
  timeSinceRefresh(){
    return this.timeSince(this.store.jobs.getLastUpdate(this.slug)) || Date.now()
  }
}

export default defineComponent({
  name: 'GetPulse',
  components: {},
  data() {
    return {
      slug: 'relays/pulse',
      pulses: {},
      interval: null,
    }
  },
  setup(){
    return { 
      store : setupStore(),
    }
  },
  created(){
    clearInterval(this.interval)
  },
  unmounted(){
    clearInterval(this.interval)
  },
  beforeMount(){
    this.lastUpdate = this.store.jobs.getLastUpdate(this.slug)
    
    this.relays = Array.from(new Set([...this.store.relays.getAll, ...relays]))
  },
  mounted(){
    console.log('ppulse?')
    this.invalidatePulse()
    // this.invalidateJob()
    // this.interval = setInterval( this.invalidateJob, 1000 )
  },
  updated(){
    
  },
  computed: Object.assign(SharedComputed, {}),
  methods: Object.assign(localMethods, RelayMethods),
  props: {},
})
</script>

<style scoped>
 #refresh { font-size: 12pt; color:#666; margin-bottom:15px }
 #refresh button { cursor: pointer; border-radius: 3px; border: 1px solid #a0a0a0; color:#333 }
 #refresh button:hover {color:#000;}
 #refresh button[disabled] {color:#999 !important; border-color:#e0e0e0}
</style>

