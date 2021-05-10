<template>
    <div class="upload">

        <div class="upload__input_div">
            <input type="text" autofocus maxlength="100" id="linkInput" class="upload__input_input" placeholder="Вставьте ссылку"
                   v-model="input"
            />
        </div>
        <button @click.prevent="checkMimeType" class="upload__button" type="submit" name="action">
            Загрузить
        </button>

    </div>

</template>

<script>
    export default {
        data() {
            return {
                input: '',
                ImagesObject: {}
            }
        },
        methods: {
            checkMimeType() {
                const img = new Image();
                img.src = this.input;
                img.onload = () => this.emitImgToGallery(img.src); //если все хорошо, то это изображение
                img.onerror = () => this.checkJSON(); //если нет, то это что-то другое, проверяем, не json ли
            },
            checkJSON() {
                fetch(this.input)
                    .then(async res => res.blob()) // получаем результат по ссылке и делаем из него blob
                    .then(blob => {
                        if (blob.type === 'application/json') this.emitJSONToGallery(); // если это json, вызываем метод
                    })
                    .catch(() => {
                        alert("Ссылка не рабочая");
                        this.input = '';
                    });
                return;
            },

            emitImgToGallery(imgSrc) {
                this.$emit('uploadIMG', imgSrc);
                this.input = '';
            },
            emitJSONToGallery() {
                this.$emit('uploadJSON', this.input);
            }


        }
    }
</script>

<style scoped>

</style>
