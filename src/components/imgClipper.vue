<template>
    <Teleport to="body">
        <div class="mask">
            <div class="main">
                <div
                    id="contain"
                    ref="contain"
                    style="display: inline-block; width: 40%"
                >
                    <img
                        v-if="imgURL"
                        :src="imgURL"
                        draggable="false"
                        style="width: 100%"
                    />
                    <div
                        id="selection"
                        :style="{
                            left: selectionPosition.leftTopX + 'px',
                            top: selectionPosition.leftTopY + 'px',
                            width: selectionPosition.width + 'px',
                            height: selectionPosition.height + 'px',
                        }"
                        ref="selection"
                        @mousedown.self="moveSelection($event, 'start')"
                        @mouseup.self="moveSelection($event, 'end')"
                    ></div>
                    <div
                        id="leftTop"
                        :style="{
                            left: hornPosition.leftTop.positionX + 'px',
                            top: hornPosition.leftTop.positionY + 'px',
                        }"
                        @mousedown.stop="moveHorn($event, 'start')"
                        @mouseup.stop="moveHorn($event, 'end')"
                    ></div>
                    <div
                        id="rightTop"
                        :style="{
                            left: hornPosition.rightTop.positionX + 'px',
                            top: hornPosition.rightTop.positionY + 'px',
                        }"
                        @mousedown.stop="moveHorn($event, 'start')"
                        @mouseup.stop="moveHorn($event, 'end')"
                    ></div>
                    <div
                        id="leftBottom"
                        :style="{
                            left: hornPosition.leftBottom.positionX + 'px',
                            top: hornPosition.leftBottom.positionY + 'px',
                        }"
                        @mousedown.stop="moveHorn($event, 'start')"
                        @mouseup.stop="moveHorn($event, 'end')"
                    ></div>
                    <div
                        id="rightBottom"
                        :style="{
                            left: hornPosition.rightBottom.positionX + 'px',
                            top: hornPosition.rightBottom.positionY + 'px',
                        }"
                        @mousedown.stop="moveHorn($event, 'start')"
                        @mouseup.stop="moveHorn($event, 'end')"
                    ></div>
                </div>
                <img
                    id="preview"
                    src=""
                    ref="preview"
                    style="display: inline-block"
                />
                <div class="btns">
                    <button @click="submit">确定</button>
                </div>
            </div>
        </div>
    </Teleport>
</template>

<script lang="ts">
    export default {
        name: "imgClipper",
    };
</script>
<script setup lang="ts">
    import { toRefs, ref, watch, reactive, computed } from "vue";
    import lodash from "lodash";

    const imgURL = ref("");
    // 裁剪完成之后的生成的Blob对象
    const clippedBlob = ref<Blob>();
    // 包含图像和调节控件的容器, 它的大小由图片撑起
    const contain = ref<HTMLDivElement>();
    // 选择范围
    const selection = ref<HTMLDivElement>();
    // 预览图片的图片元素
    const preview = ref<HTMLImageElement>();
    // 临时变量, 用于存储当前拖动的控件
    const currentHorn = ref<HTMLDivElement>();
    // 接受两个props, 要裁剪的文件和压缩质量

    const props = defineProps<{
        file: File | undefined;
        quality: number;
    }>();
    const { file, quality } = toRefs(props);
    // output事件用于向外界传递裁剪完成的数据
    const emit = defineEmits(["output"]);

    // 临时变量, 用于存储开始时的鼠标位置
    let initPosition = {
        mouseX: 0,
        mouseY: 0,
    };
    // 临时变量, 用于存储开始时的控件位置
    let initHornPosition = {} as any;
    // 四个控件的位置
    const hornPosition = reactive({
        leftTop: {
            positionX: 50,
            positionY: 50,
        },
        rightTop: {
            positionX: 150,
            positionY: 50,
        },
        leftBottom: {
            positionX: 50,
            positionY: 150,
        },
        rightBottom: {
            positionX: 150,
            positionY: 150,
        },
    });
    // 根据四个控件的位置, 计算出选择区域的左上角x, y坐标及宽高;
    const selectionPosition = computed(() => {
        const leftTopX = hornPosition.leftTop.positionX;
        const leftTopY = hornPosition.leftTop.positionY;
        const width =
            hornPosition.rightTop.positionX -
            hornPosition.leftTop.positionX +
            25;
        const height =
            hornPosition.leftBottom.positionY -
            hornPosition.leftTop.positionY +
            25;
        return {
            leftTopX,
            leftTopY,
            width,
            height,
        };
    });
    // 监听外界传进来的props file变化, 如果有变化, 就将其转换成dataUrl以便显示在预览框上
    watch(
        file,
        () => {
            if (file.value) {
                // 初始化fileReader
                let reader = new FileReader();
                reader.readAsDataURL(file.value);
                reader.onload = () => {
                    imgURL.value = reader.result as string;
                };
            }
        },
        {
            immediate: true,
        },
    );
    // 移动选择区域的事件
    const moveSelection = (e: MouseEvent, isMove: string) => {
        // 如果没有元素, 直接返回
        if (!contain.value && selection.value) {
            return;
        }
        // 记录鼠标初始位置
        initPosition = {
            mouseX: e.pageX,
            mouseY: e.pageY,
        };
        const raw = JSON.stringify(hornPosition);
        Object.assign(initHornPosition, JSON.parse(raw));
        if (isMove == "start") {
            document.addEventListener("mousemove", handelMoveSelection);
        } else {
            document.removeEventListener("mousemove", handelMoveSelection);
            throttledClip();
        }
    };
    const handelMoveSelection = (e: MouseEvent) => {
        // 计算出鼠标位置偏移量, 然后给四个角增加偏移即可
        const offsetX = e.pageX - initPosition.mouseX;
        const offsetY = e.pageY - initPosition.mouseY;

        hornPosition.leftTop.positionX =
            initHornPosition.leftTop.positionX + offsetX;
        hornPosition.rightTop.positionX =
            initHornPosition.rightTop.positionX + offsetX;
        hornPosition.leftBottom.positionX =
            initHornPosition.leftBottom.positionX + offsetX;
        hornPosition.rightBottom.positionX =
            initHornPosition.rightBottom.positionX + offsetX;

        hornPosition.leftTop.positionY =
            initHornPosition.leftTop.positionY + offsetY;
        hornPosition.rightTop.positionY =
            initHornPosition.rightTop.positionY + offsetY;
        hornPosition.leftBottom.positionY =
            initHornPosition.leftBottom.positionY + offsetY;
        hornPosition.rightBottom.positionY =
            initHornPosition.rightBottom.positionY + offsetY;
    };

    // 移动控件的事件
    const moveHorn = (e: MouseEvent, isMove: string) => {
        if (!contain.value && selection.value) {
            return;
        }
        currentHorn.value = e.target as HTMLDivElement;
        // 初始位置
        initPosition = {
            mouseX: e.offsetX,
            mouseY: e.offsetY,
        };
        const raw = JSON.stringify(hornPosition);
        Object.assign(initHornPosition, JSON.parse(raw));
        if (isMove == "start") {
            document.addEventListener("mousemove", handelMoveHorn);
        } else {
            document.removeEventListener("mousemove", handelMoveHorn);
            throttledClip();
        }
    };
    const getMousePositionInBox = (e: MouseEvent, ele: HTMLElement) => {
        const rect = ele.getBoundingClientRect();
        // getBoundingClientRect()可以返回一个元素的大小、位置信息。
        // 此函数会返回一个具有四个属性（left、top、right、bottom）
        // 的DOMRect对象，其中这些属性表示了该元素相对于可视区域左上角的坐标、宽度和高度。通过调用该方法，还可以获取元素的边框大小。
        // 该方法通常用于计算元素在网页中的位置以及处理事件时鼠标指针的位置。
        const mouseY = e.pageY - rect.top;
        const mouseX = e.pageX - rect.left;

        return {
            mouseX,
            mouseY,
        };
    };
    const handelMoveHorn = (e: MouseEvent) => {
        if (
            selectionPosition.value.height < 50 ||
            selectionPosition.value.width < 50
        ) {
            // 进行某些处理
        }
        // 先获取鼠标在contain内部的坐标
        const mousePosition = getMousePositionInBox(
            e,
            contain.value as HTMLElement,
        );

        const positionY = mousePosition.mouseY - initPosition.mouseY;
        const positionX = mousePosition.mouseX - initPosition.mouseX;
        currentHorn.value!.style.top = positionY + "px";

        currentHorn.value!.style.left = positionX + "px";

        // 更新四角的坐标
        switch (currentHorn.value?.id) {
            case "leftTop":
                hornPosition.leftTop.positionX = positionX;
                hornPosition.leftTop.positionY = positionY;

                // 更改左上角的位置同时也会影响左下角和右上角
                hornPosition.leftBottom.positionX = positionX;
                hornPosition.rightTop.positionY = positionY;

                break;
            case "rightTop":
                hornPosition.rightTop.positionX = positionX;
                hornPosition.rightTop.positionY = positionY;

                // 更改右上角的位置同时也会影响左上角和右下角
                hornPosition.leftTop.positionY = positionY;
                hornPosition.rightBottom.positionX = positionX;

                break;
            case "leftBottom":
                hornPosition.leftBottom.positionX = positionX;
                hornPosition.leftBottom.positionY = positionY;

                // 更改左下角的位置同时也会影响左上角和右下角
                hornPosition.leftTop.positionX = positionX;
                hornPosition.rightBottom.positionY = positionY;
                break;
            case "rightBottom":
                hornPosition.rightBottom.positionX = positionX;
                hornPosition.rightBottom.positionY = positionY;

                // 更改右下角的位置同时也会影响右上角和左下角
                hornPosition.rightTop.positionX = positionX;
                hornPosition.leftBottom.positionY = positionY;
                break;
        }
    };
    const throttledClip = lodash.throttle(() => {
        clip();
    }, 16);
    const clip = async () => {
        // 先将文件转换成img,然后转换成blob
        if (file.value) {
            const img = await fileToImg(file.value);
            // 因为展示的图片是固定的宽度, 所以要计算一个放大的比例
            const rate = img.width / contain.value!.clientWidth;
            const clippedImg = await imgToClippedImg(
                img,
                selectionPosition.value.leftTopX * rate,
                selectionPosition.value.leftTopY * rate,
                selectionPosition.value.width * rate,
                selectionPosition.value.height * rate,
            );
            const blob = await imgToBlobCompress(
                clippedImg,
                1000,
                1000,
                quality.value,
                "image/png",
            );
            clippedBlob.value = blob;
            const reader = new FileReader();
            reader.readAsDataURL(blob);
            reader.onload = () => {
                if (preview.value && typeof reader.result == "string") {
                    preview.value.src = reader.result;
                    preview.value.width = selectionPosition.value.width;
                }
            };
        }
    };
    // 传入一个file, 异步返回一个image对象
    const fileToImg = (file: File): Promise<HTMLImageElement> => {
        return new Promise((resolve, reject) => {
            const reader = new FileReader();
            const img = new Image();
            // 将传进来的file转换成dataURl
            reader.readAsDataURL(file);
            reader.onload = () => {
                img.src = reader.result as string;
            };
            reader.onerror = e => {
                reject(e);
            };
            img.onload = () => {
                resolve(img);
            };
            img.onerror = e => {
                reject(e);
            };
        });
    };
    // 传入image, 返回裁剪后的image
    const imgToClippedImg = (
        img: HTMLImageElement,
        dx: number,
        dy: number,
        width: number,
        height: number,
    ): Promise<HTMLImageElement> => {
        return new Promise(resolve => {
            // 这个函数用来把传入的图片裁剪, 返回一个img
            const canvas = document.createElement("canvas");
            const context = canvas.getContext("2d");
            canvas.width = width;
            canvas.height = height;
            context?.drawImage(img, dx, dy, width, height, 0, 0, width, height);
            const clipDataUrl = canvas.toDataURL("image/webp", quality.value);
            const image = new Image();
            image.src = clipDataUrl;
            image.onload = () => {
                resolve(image);
            };
        });
    };
    // 传入image, 返回压缩后的Blob对象
    const imgToBlobCompress = (
        img: HTMLImageElement,
        maxWidth: number,
        maxHeight: number,
        quality: number,
        type: "image/png" | "image/webp",
    ): Promise<Blob> => {
        return new Promise((resolve, reject) => {
            const canvas = document.createElement("canvas");
            const ctx = canvas.getContext("2d");

            // 获取图片原始尺寸
            const { width: originWidth, height: originHeight } = img;
            // 确定目标尺寸
            let targetWidth = originWidth,
                targetHeight = originHeight;
            // 如果图片尺寸大于目标尺寸, 那么就进行等比缩放
            if (targetWidth > maxWidth || targetHeight > maxHeight) {
                // 等比缩小的算法
                // 如果是横着的图, 图片的宽等于最大宽度, 高等比缩放;
                // 如果是竖着的图, 图片的高等于最大高度, 宽等比缩放;
                // 先判断是横着的图还是竖着的图
                if (targetWidth > targetHeight) {
                    //横着的
                    targetWidth = maxWidth;
                    targetHeight = targetHeight * (maxWidth / originWidth);
                } else {
                    //竖着的
                    targetHeight = maxHeight;
                    targetWidth = targetWidth * (maxHeight / originHeight);
                }
            }
            try {
                // 把img绘制进canvas
                canvas.width = targetWidth;
                canvas.height = targetHeight;
                ctx?.clearRect(0, 0, targetWidth, targetHeight);
                ctx?.drawImage(img, 0, 0, targetWidth, targetHeight);
                // 获取blob对象
                canvas.toBlob(
                    blob => {
                        if (blob) {
                            resolve(blob);
                        } else {
                            reject(new Error("转换失败"));
                        }
                    },
                    type,
                    quality,
                );
            } catch (error) {
                reject(error);
            }
        });
    };
    const submit = () => {
        emit("output", clippedBlob.value);
    };
    clip();
    document.addEventListener("mouseup", () => {
        document.removeEventListener("mousemove", handelMoveSelection);
    });
    document.addEventListener("mouseup", () => {
        document.removeEventListener("mousemove", handelMoveHorn);
    });
</script>

<style scoped lang="less">
    .mask {
        z-index: 999;
        display: flex;
        position: absolute;
        top: 0;
        left: 0;
        right: 0;
        bottom: 0;
        background-color: rgba(73, 73, 73, 0.685);
    }
    .main {
        position: relative;
        width: 80vw;
        background-color: #fff;
        margin: auto;
        border-radius: 30px;
        box-shadow: #00000065 0 0 50px 20px;
    }
    #contain {
        user-select: none;
        position: relative;
        margin: 20px;
        overflow: hidden;
        #selection {
            cursor: move;
            background-color: #5f5f5f17;
            position: absolute;
            box-shadow: #0000008a 0 0 0 10000px;
        }
        #leftTop {
            cursor: nw-resize;
            position: absolute;
            // left: 25%;
            // top: 25%;
            width: 20px;
            height: 20px;
            border-left: 5px solid #ffffff;
            border-top: 5px solid #ffffff;
            mix-blend-mode: difference;
        }
        #rightTop {
            cursor: ne-resize;
            position: absolute;
            // left: 75%;
            // top: 25%;
            width: 20px;
            height: 20px;
            border-right: 5px solid #ffffff;
            border-top: 5px solid #ffffff;
            mix-blend-mode: difference;
        }
        #leftBottom {
            cursor: ne-resize;
            position: absolute;
            // left: 25%;
            // top: 75%;
            width: 20px;
            height: 20px;
            border-left: 5px solid #ffffff;
            border-bottom: 5px solid #ffffff;
            mix-blend-mode: difference;
        }
        #rightBottom {
            cursor: nw-resize;
            position: absolute;
            // left: 75%;
            // top: 75%;
            width: 20px;
            height: 20px;
            border-right: 5px solid #ffffff;
            border-bottom: 5px solid #ffffff;
            mix-blend-mode: difference;
        }
    }
    #preview {
        margin: 20px;
    }
    .btns {
        position: absolute;
        top: 5%;
        right: 5%;
        button {
            width: 5.12vmax;
            height: 3.65vmax;
            margin: 1vmax;
            border: 0;
            border-radius: 1.82vmax;
            color: white;
            background-color: #1a8cd8;
            font-size: 1.62vmax;
        }
    }
</style>
