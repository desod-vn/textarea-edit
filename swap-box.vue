<script>
export default {
    data() {
        return {
            startSelection: 0,
            showArea: true,
            text: '',
            textShow: '',
            before: '',
            after: '',
            isDelete: false,
            tmpPosition: null,
            backNormal: false,
        }
    },
    props: {
        default: {
            type: String,
            default: ''
        },
        id: {
            required: true,
            type: String,
        },
        tagList: {
            type: Array,
            required: true
        },
        tag: {
            type: String,
            default: ''
        },
        limit: {
            type: Number,
            default: 1000
        }
    },
    watch: {
        text() {
            this.textShow = this.replaceWithTag(this.text)
            if(this.backNormal) {

            }
        },
        tag() {
            if(!!this.tag) {
                this.addTag(this.tag)
            }
        }
    },
    methods: {
        addTag(content) {
            const ins = `[#${content}]`;
            if (this.text.length > this.limit) {
                return;
            }
            const before = this.text.substring(0, this.startSelection);
            const after = this.text.substring(this.startSelection);
            if (!this.isInATag(before)) {
                return
            };
            this.text = before + ins + after;
            this.startSelection = this.startSelection + ins.length;
            this.turnOffShow(null, this.startSelection, this.startSelection)
        },
        currentPointer(e) {
            this.startSelection = e.target.selectionStart;
            this.$emit('currentPosition', this.selectionStart)
            this.deleteOnPress(e);
        },
        deleteOnPress(e) {
            const tagKey = this.tagList.map(x => x.key);
            if (e.type === 'keydown') {
                this.before = this.text.substring(0, this.startSelection);
                this.after = this.text.substring(this.startSelection);
                if (e.keyCode == 8 && this.before[this.before.length - 1] == ']') {
                    const lastBeforeOpen = this.before.lastIndexOf('[#');
                    const lastBeforeClose = this.before.lastIndexOf(']');
                    const selectedTag = this.text.slice(lastBeforeOpen, lastBeforeClose + 1)
                    this.before = this.before.slice(0, -1 * selectedTag.length)
                    this.tmpPosition = lastBeforeOpen;
                    this.isDelete = true
                } else if (e.keyCode == 46 && this.after[0] == '[' && this.after[1] == '#') {
                    const lastAfterOpen = this.after.indexOf('[#');
                    const lastAfterClose = this.after.indexOf(']');
                    const selectedTag = this.text.slice(lastAfterOpen, lastAfterClose + 1)
                    this.after = this.after.slice(selectedTag.length)
                    this.tmpPosition = this.before.length + lastAfterOpen;
                    this.isDelete = true
                }
            }
            if (e.type === 'keyup' && this.isDelete) {
                this.text = this.before + this.after
                this.isDelete = false;
                this.turnOffShow(null, this.tmpPosition, this.tmpPosition)
            }
        },
        turnOffShow(e, startPos = -1, endPod = -1) {
            if (!!e && e.target.getAttribute('data-value') == 'close') {
                return this.handleFilterTextInArea(e);
            }
            this.showArea = false;
            this.$nextTick(() => {
                this.$refs.input.focus();
                this.$refs.input.setSelectionRange(startPos, endPod); //END OF INPUT
            })
        },
        turnOnShow() {
            this.showArea = true;
            this.$emit('throwText', this.text)
        },
        replaceWithTag(text) {
            let out = text;
            const classSpan = `
                margin: 1px;
                color: #616161;
                background: #ECEEF1;
                border-radius:4px;
                padding: 0px 8px;
                cursor: pointer;
                word-wrap: break-word;
                display: inline-block;
                font: normal normal normal 12px/17px Noto Sans CJK JP;
            `;

            const textRemoveClass = `
                class="material-icons-round"
                data-value="close"
                style="
                    font-size: 14px;
                    transform:translateY(3px);
                "
            `;

            const tagExistNumber = text.split('[#').length - 1
            let i = 0;
            do {
                let key = Math.random();
                const { result, isValidTag, selectedTag } = this.canSwapTag(out);
                if(result && isValidTag) {
                    out = out.replace('[#', `<span style="${classSpan}" id="${key}">`);
                    const tag = this.tagList.find(x => x.key === selectedTag)
                    out = out.replace(selectedTag, tag.label);
                    out = out.replace(']', `<span ${textRemoveClass}>close</span></span>`);
                } else if (!result && isValidTag) {
                    out = out.replace(`[#${selectedTag}]`, '')
                    this.text = this.text.replace(`[#${selectedTag}]`, selectedTag)
                    this.turnOffShow(null, this.startSelection - 1, this.startSelection - 1)
                    continue;
                }
                i++;
            } while (i < tagExistNumber);
            return out;
        },
        canSwapTag(text) {
            const firstOpen = text.indexOf('[#');
            const firstClose = text.indexOf(']');
            const tagKey = this.tagList.map(x => x.key);
            let result = true;
            let selectedTag = null;
            let isValidTag = true;
            if (firstClose + firstOpen <= 0) {
                result = false;
                isValidTag = false;
            }
            if (firstClose < firstOpen) {
                result = false;
                isValidTag = false;
            }
            selectedTag = text.slice(firstOpen + 2, firstClose)
            if (!tagKey.includes(selectedTag.trim())) {
                result = false
            }
            return {
                result: result,
                isValidTag: isValidTag,
                selectedTag: selectedTag
            };
        },
        isInATag(before) {
            const lastBeforeOpen = before.lastIndexOf('[#');
            const lastBeforeClose = before.lastIndexOf(']');
            if (lastBeforeClose < lastBeforeOpen) {
                return false;
            }
            return true;
        },
        handleFilterTextInArea(e) {
            let content = '';
            const listElm = document.getElementById(this.id).childNodes
            const idParent = e.target.parentNode.id
            for (let i = 0; i < listElm.length; i++) {
                if (listElm[i].nodeName == 'SPAN' && idParent != listElm[i].id) {
                    const textTag = listElm[i].firstChild.textContent;
                    const tag = this.tagList.find(x => x.label === textTag)
                    const key = tag.key;
                    content += `[#${key}]`;
                } else if (listElm[i].nodeName == '#text') {
                    content += listElm[i].textContent;
                }
            }
            return this.text = content
        },
    },
    mounted() {
        this.text = this.default;
        ['focus', 'keyup', 'mouseup', 'mousedown', 'keydown'].forEach(evt => {
            this.$refs.input.addEventListener(evt, this.currentPointer)
        })
    },
    beforeUnmount() {
        ['focus', 'keyup', 'mouseup', 'mousedown', 'keydown'].forEach(evt => {
            this.$refs.input.removeEventListener(evt, this.currentPointer)
        });
    }
}
</script>
<template>
    <textarea
        v-model="text"
        v-show="!showArea"
        class="box__textarea w-100"
        ref="input"
        @focusout="turnOnShow"
        spellcheck="false"
    />
    <div
        v-show="showArea"
        :id="id"
        class="box__textarea w-100 whitespace-pre-wrap cursor-text"
        @click="turnOffShow"
        v-html="textShow"
    ></div>
</template>
<style scoped>
.box__textarea:focus {
    outline: none;
    box-shadow: inset 0 0 0 #ddd;
}
.box__textarea, .box__textarea:focus {
    background: #FFFFFF 0% 0% no-repeat padding-box;
    border: 1px solid #CAD1D7;
    border-radius: 4px;
    padding: 5px 8px;
    text-align: left;
    font: normal normal normal 12px/17px Noto Sans CJK JP;
    letter-spacing: 0px;
    color: #616161;
    overflow-y: scroll;
    display: block;
    margin: 0;
    max-height: 236px;
    box-sizing: border-box;
    height: 236px;
}

.box__textarea::-webkit-scrollbar, .box__input::-webkit-scrollbar {
    display: none;
}

.box__textarea:focus , .box__input:focus {
    outline: none;
}

</style>
