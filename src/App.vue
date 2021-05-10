<template>
    <div id="app" class="container" >
        <Gallery/>
        <div id="disclaimer" class="disclaimer">
            <h1>Управление галереей</h1>
            <ul>
                <li>
                    Вставьте в поле ссылку на изображение и нажмите "Загрузить" или
                    перетащите в поле галереи изображение (или несколько) с компьютера. <br>
                    Изображение добавится в галерею.<br>
                    Файлы, не являющееся изображениями, проигнорируются.<br>
                </li>
                <li>
                    Вставьте в поле ссылку на JSON файл, возвращающий объект, содержащий массив объектов
                    со свойством "url" (адреса изображений),
                    например:<br>
                    <a href="https://don16obqbay2c.cloudfront.net/frontend-test-task/gallery-images.json">
                        https://don16obqbay2c.cloudfront.net/frontend-test-task/gallery-images.json</a><br>
                    и нажмите "Загрузить". <br>
                    Или перетащите его в поле галереи.<br>
                    Галерея заполнится картинками.
                </li>
                <li>
                    Для удаления картинки из галереи перетащите ее за границы контейнера в подсвечиваемую область.
                    На мобильном устройстве - задержите палец на картинке и свайпните или так же перетащите за границы контейнера.
                </li>

            </ul>
        </div>
    </div>
</template>

<script>
    import Gallery from "@/components/Gallery";

    export default {
        name: 'App',
        components: {
            Gallery
        }
    }
</script>

