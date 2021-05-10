<template>
    <div ref="all">
        <div class="overlay"
             :class="{ danger: danger,transparent:ready }"
             @drop="removePic($event.dataTransfer.getData('index'))"
             @dragover.prevent
             @dragleave="ready=false"
             @dragenter="danger=innerEvent">

        </div>
        <div class="gallery" ref="gallery"
             @dragenter="addRemoveClasses()"
             :class="{ready:ready}">
            <h1>VueGallery</h1>

            <GalleryUpload @uploadJSON="uploadJSON" @uploadIMG="prepareImg"/>

            <div class="gallery__block" ref="gallery_size" id="galleryBlock">
                <div v-if="(rows.length>0)" class="gallery__block">

                    <div draggable="true"
                         @dragstart="onDragStart($event,index)"
                         @dragend="showDisclaimer()"
                         @touchstart="setTouchTimer($event)"
                         @touchmove="touchMove($event)"
                         @touchend="touchEnd()"
                         @touchcancel="touchCancel()"
                         class="gallery__block_container"
                         v-for="(pic,index) of galleryImages"
                         :gen="pic.gen"
                         :key="index"
                         :id="index"
                         :style="{width:rows[index].widthPerCent+'%',height:'auto'}">
                        <img class="gallery__block_img" :src="pic.url" alt=""
                             width="100%" height="100%"
                             :id="'img'+index"/>
                    </div>
                </div>
            </div>
        </div>
    </div>
</template>

<script>
    import config from '@/assets/config.scss'; //импортируем переменные из sass
    import GalleryUpload from '@/components/GalleryUpload'; // импортруем компонент аплоада

    export default {
        data() {
            return {
                galleryImages: [],                      //рабочий массив картинок
                contWidthMax: config.contWidthMax,      //устанавливаем макимальную ширину галереи(892px)
                contWidthMin: config.contWidthMin,      //устанавливаем минимальную ширину галереи(320px)
                colsMax: config.colsMax,                //устанавливаем макимальное количество колонок(5)
                colsMin: config.colsMin,                //устанавливаем минимальное количество колонок(1)
                imageMin: config.imageMin,              //устанавливаем минимальную ширину картинки(100px)
                contWidth: 0,                           //ширина контейнера
                tempArray: [],                          //временный массив объектов
                rows: [],                               //массив размеров картинок
                danger: false,           //для добавления класса
                ready: false,            //для добавления класса
                innerEvent: false,       //для проверки источника drag&drop
                ddEvents: ['dragend', 'dragover', 'dragenter', 'dragleave', 'drop'],         //перечисляем события для запрещения действия по умолчанию
                touchableElement: {},   //перетаскиваемый тачем элемент
                touchXposition: 0,   //текущая позиция тача X
                touchYposition: 0,  //текущая позиция тача Y
                touchElY: 0,    //координаты тача от края элемента
                touchElX: 0,    //координаты тача от края элемента
                touchStartX: 0,     //стартовая позиция тача X
                touchStartY: 0,     //стартовая позиция тача Y
                galleryTop: 0,      //координаты блока галереи
                galleryBottom: 0,
                galleryLeft: 0,
                galleryRight: 0,
                touchduration: config.touchduration, //время задержки тача
                timer: 0,              // номер таймера
                lockTimer: false,       // флаг блокировки
                touchStarted: false,     // флаг начавшегося тача
                touchStartMoveTime: 0 //время старта движения тача

            };
        },
        components: {
            GalleryUpload
        },
        mounted() {
            for (let i = 0; i < 3; i++) {
                this.addDefaultPics();
            }
            this.promiceContWidth(); //замеряем ширину галереи
            document.getElementsByClassName('gallery__block_img').ondragstart = function () {       //запрещаем перетаскивать изображения
                return false;
            };
            window.addEventListener('resize', this.promiceContWidth, {passive: true});          //добавляем "слушателя" события resize
            this.notNativeEvents();
            this.onGalleryDrop();
        },
        beforeDestroy() {
            window.removeEventListener('resize', this.setContWidth);        // удаляем слушателя (тут необходимости нет, но для расширения пригодится)
        },
        computed: {
            currentArea: function () {      //вычисляем текущую область при изменении ширины галереи
                let currentArea = 0;
                for (let i = 0; i <= Math.round((this.contWidthMax - this.contWidthMin) / this.imageMin); i++) {        //отслеживаемая область равна минимальной ширине изображения
                    if ((this.contWidth >= this.contWidthMax - this.imageMin * i) && (this.contWidth < this.contWidthMax - this.imageMin * (i - 1))) {
                        currentArea = i;
                    }
                }
                return currentArea;
            }


        },
        watch: {
            currentArea: function () {
                this.startCalculate();          //запускаем расчет размеров изображений при изменении области (если ширина галереи изменилась на мин. ширину изображения)
            },
            galleryImages: function () {        //отслеживаем изменение основного массива объектов изображений
                this.danger = false;        //убираем класс подсветки области "удаления"
                this.ready = false;         //убираем класс ожидания
                this.startCalculate(); //запускаем расчет размеров изображений при изменении
                this.galleryCoords();
            }
        },
        methods: {
            addDefaultPics() {
                let url = 'images/no_image.jpg';
                this.uploadIMG(url, 640, 426, 'default');
            },
            removeDefaults() {
                if (this.galleryImages[0].gen === 'default') {
                    this.galleryImages.splice(0, 1);
                }

            },
            notNativeEvents() {
                this.ddEvents.forEach(function (evt) {
                    this.$refs.all.addEventListener(evt, function (e) { //запрещаем ненужные дефолтные методы браузера для всего документа
                        e.preventDefault();
                        e.stopPropagation();
                    }.bind(this), false);
                }.bind(this));
            },
            onGalleryDrop() {
                this.$refs.gallery.addEventListener('drop', e => this.insertToGallery(e), false);
                this.innerEvent = false; //заканчиваем внутренний drag&drop
                this.ready = false; //меняем флаг с ожидания на обычное состояние
            },
            insertToGallery(e) {
                let files = e.dataTransfer.files;
                if (files.length > 0) {
                    for (let i = 0; i < files.length; i++) {        //перебираем массив файлов
                        const type = files[i].type;
                        if (type.replace(/\/.+/, '') === 'image') {     // если тип - изображение
                            let src = URL.createObjectURL(files[i]);
                            this.prepareImg(src);
                        } else if (type === 'application/json') {      //если тип - json, вызываем метод для парсинга json
                            this.parseJSON(URL.createObjectURL(files[i]));
                        } else {
                            alert('Только изображения или JSON-файл'); // выкидываем предупреждение
                        }
                    }

                }

            },
            uploadJSON(data) {
                this.parseJSON(data);   //парсим полученный из инпута JSON
            },
            parseJSON(json) {       //парсим JSON file
                fetch(json)
                    .then(async response => {
                        const jsonObject = await response.json();
                        if (!response.ok) {
                            const error = (jsonObject && jsonObject.message) || response.statusText;
                            return Promise.reject(error);
                        }
                        let errorElements = [];
                        let urls = [];

                        for (let key in jsonObject) {
                            for (let i = 0; i < jsonObject[key].length; i++) {
                                if (!jsonObject[key][i].url) {
                                    errorElements.push(i);
                                } else {
                                    urls.push(jsonObject[key][i].url);
                                }
                            }

                        }

                        if (urls.length > 0) {
                            this.galleryImages = []; //чистим галерею от предыдущих картинок
                            for (let i = 0; i < urls.length; i++) {
                                await this.prepareImg(urls[i]);           //отправляем на добавление картинок

                            }

                        } else {
                            return Promise.reject('В полученном файле не найдено ни одного url')
                        }

                        if (errorElements.length > 0) {
                            return Promise.reject('В полученном файле для некоторых записей не указаны url')
                        }


                    })
                    .catch((reject) => {
                        alert(reject);
                    });
            },
            prepareImg(src) {
                const img = new Image();
                img.src = src;
                img.onload = () => this.uploadIMG(img.src, img.width, img.height, 'received'); //вызываем метод добавления изображения


            },
            uploadIMG(imgSrc, imgWidth, imgHeight, gen) {        // добавляем в галерею загруженные или перетащенные изображения
                if (gen !== 'default') {
                    this.galleryImages.push({
                        "url": imgSrc,
                        "width": imgWidth,
                        "height": imgHeight,
                        "gen": gen
                    });
                    this.removeDefaults();
                } else {
                    this.galleryImages.unshift({
                        "url": imgSrc,
                        "width": imgWidth,
                        "height": imgHeight,
                        "gen": gen
                    });
                }
            },
            galleryCoords() {
                let coords = document.getElementById('galleryBlock').getBoundingClientRect()
                this.galleryTop = coords.top;
                this.galleryLeft = coords.left;
                this.galleryBottom = coords.bottom;
                this.galleryRight = coords.right;
            },
            onDragStart(event, index) {     //определяем свои события drag&drop
                this.innerEvent = true;
                event.dataTransfer.dropEffect = 'move';
                event.dataTransfer.effectAllowed = 'move';
                event.dataTransfer.setData('index', index);
                event.target.classList.add('draggable')
                this.hideDisclaimer();
            },
            setTouchTimer(event) {
                if (this.lockTimer) return; // если активна блокировка, возвращаем
                if (event.targetTouches.length === 1) {
                    this.timer = setTimeout(() => this.touchStart(event), this.touchduration); // запускаем таймер до запуска старта тача
                    this.lockTimer = true;
                }
            },
            touchStart(event) {
                event.preventDefault() // отключаем реакцию элементов по умолчанию в браузере
                this.touchStarted = true; // устанавливаем флаг старта тача
                this.innerEvent = true; // устанавливаем флаг внутреннего события
                this.hideDisclaimer();  // скрываем дисклеймер
                let touch = event.targetTouches[0]; // получаем объект перетаскиваемого элемента
                this.touchableElement = touch.target; // получаем перетаскиваемый элемент
                this.touchStartX = touch.pageX;
                this.touchStartY = touch.pageY;
                this.touchElX = touch.pageX - touch.target.offsetLeft; // получаем расстояние от тача до левого края элемента
                this.touchElY = touch.pageY - touch.target.offsetTop; // получаем расстояние от тача до верхнего края элемента
                this.touchXposition = touch.pageX; //записываем координаты тача
                this.touchYposition = touch.pageY; //записываем координаты тача
                touch.target.classList.add('touchable'); //добавляем класс элементу
                this.touchableElement.style.left = this.touchXposition - this.touchElX + 'px'; //устанавливаем координаты объекта
                this.touchableElement.style.top = this.touchYposition - this.touchElY + 'px'; //устанавливаем координаты объекта
            },
            touchMove(event) {
                event.preventDefault() // отключаем реакцию элементов по умолчанию в браузере
                this.touchStartMoveTime = new Date().getTime();
                if (!this.touchStarted) return; // отсекаем посторонние тачмувы
                let touch = event.targetTouches[0];
                // позиционируем перетаскиваемый элемент
                this.touchXposition = touch.pageX;
                this.touchYposition = touch.pageY;
                this.touchableElement.style.left = this.touchXposition - this.touchElX + 'px';
                this.touchableElement.style.top = this.touchYposition - this.touchElY + 'px';
                if ((this.touchXposition < this.galleryLeft) ||
                    (this.touchYposition < this.galleryTop) ||
                    (this.touchXposition > this.galleryRight) ||
                    (this.touchYposition > this.galleryBottom)) {
                    this.danger = true; //подсвечиваем область удаления
                } else {
                    this.danger = false;
                }

            },
            touchEnd() {
                this.clearTimer(); // чистим таймер
                let touchEndTime = new Date().getTime(); // получаем время окончания перетаскивания
                this.touchStarted = false; // чистим флаг начала перетаскивания
                if (this.touchableElement.id > -1) {
                    this.clearTouchable();

                    if ((
                        ((touchEndTime - this.touchStartMoveTime) < 100) &&                 // время тача меньше 100мс
                        ((Math.abs(this.touchXposition - this.touchStartX) > 100) ||     // перемещение элемента более чем на 100px
                        (Math.abs(this.touchYposition - this.touchStartY) > 100))) ||   // перемещение элемента элемента более чем на 100px
                        ((this.touchXposition < this.galleryLeft) ||        // перемещение элемента за пределы галереи
                            (this.touchYposition < this.galleryTop) ||
                            (this.touchXposition > this.galleryRight) ||
                            (this.touchYposition > this.galleryBottom))) {
                        this.removePic(this.touchableElement.id);   // удаляем элемент
                    } else {
                        this.danger = false;
                    }

                }
            },
            touchCancel() {
                this.clearTimer(); // чистим таймер
                if (!this.touchStarted) return;
                this.danger = false;
                this.clearTouchable();
            },
            clearTimer() {
                clearTimeout(this.timer);
                this.lockTimer = false;
                this.touchStarted = false;
            },
            clearTouchable(){
                this.touchableElement.style.left = '';
                this.touchableElement.style.top = '';
                this.touchableElement.classList.remove('touchable'); // сбрасываем стили и класс у тач-элемента
                this.showDisclaimer(); //показываем дисклеймер
            },
            addRemoveClasses() {
                this.danger = false; // убираем подсветку у поля удаления
                this.ready = !this.innerEvent; // выделяем галерею, если изображение перетягивается снаружи

            },
            async removePic(index) {     //удаляем изображение из массива
                this.ready = false; //меняем флаг с ожидания на обычное состояние
                if (this.innerEvent) {
                    // this.imagesBuffer.push(this.galleryImages[index]);  // при желании сначала его можно где-нибудь сохранить (добавьте переменную imagesBuffer в data)
                    await this.galleryImages.splice(index, 1);
                    if (this.galleryImages.length < 3) {
                        await this.addDefaultPics();
                    }
                    this.innerEvent = false;
                } else {
                    this.danger = false;
                }

            },
            async promiceContWidth() {
                this.contWidth = this.$refs.gallery_size.clientWidth; //получаем ширину контейнера галереи
                await this.startCalculate(); //затем запускаем расчет размеров изображений
            },
            startCalculate() {      //запускаем вычисление размеров изображений
                this.rows = [];     //обнуляем массив размеров
                this.tempArray = [...this.galleryImages];       //делаем копию основного массива
                this.getAllRows();
            },
            getAllRows() {
                while (this.tempArray.length > 0) { //пока массив объектов картинок не пуст
                    this.rows = this.rows.concat(this.getRowArray()); // заполняем массив строк
                }

            },
            getRowArray() { // получаем массив картинок в строке
                let rowArray = this.getRow(); // получаем массив картинок, заполняющих строку, выровненных по высоте
                let min = Math.min(...rowArray.map(el => el.width)); // получаем мин. ширину картинки в строке

                for (let i = this.colsMax; i >= this.colsMin; i--) {
                    rowArray = this.getRow(i);
                    min = Math.min(...rowArray.map(el => el.width)); // получаем мин. ширину картинки в строке
                    if (min >= this.imageMin) {
                        break;
                    }

                }

                if ((this.tempArray.length === rowArray.length) && (rowArray[0].height > 250)) {
                    rowArray = this.getFixRow(); // последнюю строку считаем иначе
                }
                this.tempArray.splice(0, rowArray.length); // удаляем из массива выбранные картинки
                return rowArray;

            },
            getRow(cols) {
                let [sizes, images, height, sumRatio] = [[], [...this.tempArray], 0, 0];
                if (images.length > 1) { //если больше 1 картинки в строке
                    // считаем сумму пропорций картинок, в зависимости от количества столбцов
                    for (let i = 0; i < cols && i < this.tempArray.length; i++) {
                        images[i].ratio = images[i].width / images[i].height;
                        sumRatio += images[i].ratio;
                    }
                    //получаем размеры картинок, зависимые от ширины контейнера, количества ячеек
                    for (let i = 0; i < cols && i < this.tempArray.length; i++) {
                        height = this.contWidth / sumRatio; // общая высота (высота строки)
                        let widthPerCent = (images[i].ratio / sumRatio) * 100; // ширина в процентах от контейнера
                        let width = height * images[i].width / images[i].height;
                        sizes[i] = {widthPerCent, width, height};
                    }
                } else { //если  1 картинка в строке, ограничиваем ее по высоте (не более половины ширины контейнера, и не более 250 px)
                    let height = (this.contWidth / 2 > 250) ? 250 : this.contWidth / 2;
                    let width = height * images[0].width / images[0].height;
                    let widthPerCent = width / this.contWidth * 100;
                    sizes[0] = {widthPerCent, width, height};
                }
                return sizes;
            },
            getFixRow() {
                let height = (this.contWidth / 2 > 250) ? 250 : this.contWidth / 2;
                let images = [...this.tempArray];
                let sizes = [];

                for (let i = 0; i < this.tempArray.length; i++) {
                    let width = height * images[i].width / images[i].height;

                    let widthPerCent = width / this.contWidth * 100;
                    sizes[i] = {widthPerCent, width, height};

                }

                return sizes;

            },
            hideDisclaimer() {
                document.getElementById('disclaimer').style.display = 'none';
            },
            showDisclaimer() {
                document.getElementById('disclaimer').style.display = 'block';
            }
        }
    }
</script>

