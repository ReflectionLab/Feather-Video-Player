<script setup lang="ts">
import FeatherVideoPlayer from "../packages";

const url = {
	urls: [
		"http://tx.flv.huya.com/src/1394575534-1394575534-5989656310331736064-2789274524-10057-A-0-1.flv?wsSecret=a761ad1266d58d6efe30e1d9a9bdd2ce&wsTime=6466f1c6&u=0&seqid=16843817727990000&t=103",
  ],
}
</script>

<template>
  <div>
    <feather-video-player :url="url" />
  </div>
</template>

<style scoped>

</style>

