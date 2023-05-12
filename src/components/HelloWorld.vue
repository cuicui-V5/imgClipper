<template>
    <div>图片裁剪器demo</div>
    <input
        type="file"
        accept="image/*"
        @change="readFile"
    />
    <imgClipper
        @output="getRes"
        :file="imgFile"
        :quality="0.8"
        v-if="isShow"
    ></imgClipper>
</template>

<script lang="ts">
    export default {
        name: "",
    };
</script>
<script setup lang="ts">
    import imgClipper from "./imgClipper.vue";
    import { ref } from "vue";
    const isShow = ref(false);
    let imgFile = ref<File>();
    const readFile = (e: Event) => {
        // 获取到文件
        const fileInput = e.target as HTMLInputElement;
        if (!fileInput.files) {
            return;
        }
        imgFile.value = fileInput.files[0];
        isShow.value = true;
    };
    const getRes = (e: any) => {
        console.log(e);
        isShow.value = false;
    };
</script>

<style scoped lang="less"></style>
