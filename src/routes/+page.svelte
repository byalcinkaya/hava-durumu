<script>
    import { onMount } from 'svelte';

    let cityName = '';
    let weatherData = null;
    let forecastData = [];
    let errorMessage = '';
    let isLoading = false;
    let deferredPrompt;
    let showInstallModal = false;
    let suggestion = '';

    const apiKey = '342c5b58440e975f80a87aa6cc8529e8';
    const url = 'https://api.openweathermap.org/data/2.5';

    onMount(() => {
        getLocation();

        window.addEventListener('beforeinstallprompt', (e) => {
            e.preventDefault();
            deferredPrompt = e;
            showInstallModal = true;
        });
    });

    const getLocation = () => {
        if (navigator.geolocation) {
            navigator.geolocation.getCurrentPosition(
                position => {
                    const { latitude, longitude } = position.coords;
                    getWeatherByCoords(latitude, longitude);
                },
                error => {
                    errorMessage = 'Konum izni verilmedi veya alınamadı.';
                }
            );
        } else {
            errorMessage = 'Tarayıcınız konum servisini desteklemiyor.';
        }
    };

    const getWeatherByCoords = async (lat, lon) => {
        errorMessage = '';
        isLoading = true;
        try {
            const weatherResponse = await fetch(
                `${url}/weather?lat=${lat}&lon=${lon}&appid=${apiKey}&units=metric&lang=tr`
            );
            if (!weatherResponse.ok) {
                throw new Error('Hava durumu verileri alınamadı.');
            }
            weatherData = await weatherResponse.json();
            getSuggestions(weatherData); // Öneri oluşturma

            const forecastResponse = await fetch(
                `${url}/forecast?lat=${lat}&lon=${lon}&appid=${apiKey}&units=metric&lang=tr`
            );
            if (!forecastResponse.ok) {
                throw new Error('Hava durumu tahmin verileri alınamadı.');
            }
            const forecastJson = await forecastResponse.json();
            processForecastData(forecastJson);

            cityName = '';
        } catch (error) {
            errorMessage = error.message;
            weatherData = null;
            forecastData = [];
        } finally {
            isLoading = false;
        }
    };

    const getResult = async () => {
        if (!cityName) {
            errorMessage = 'Lütfen bir şehir giriniz.';
            return;
        }
        errorMessage = '';
        isLoading = true;
        try {
            const weatherResponse = await fetch(
                `${url}/weather?q=${cityName}&appid=${apiKey}&units=metric&lang=tr`
            );
            if (!weatherResponse.ok) {
                throw new Error('Şehir bulunamadı.');
            }
            weatherData = await weatherResponse.json();
            getSuggestions(weatherData); // Öneri oluşturma

            const forecastResponse = await fetch(
                `${url}/forecast?q=${cityName}&appid=${apiKey}&units=metric&lang=tr`
            );
            if (!forecastResponse.ok) {
                throw new Error('Hava durumu tahmin verileri alınamadı.');
            }
            const forecastJson = await forecastResponse.json();
            processForecastData(forecastJson);

        } catch (error) {
            errorMessage = error.message;
            weatherData = null;
            forecastData = [];
        } finally {
            isLoading = false;
        }
    };

    const processForecastData = (data) => {
        forecastData = [];

        const today = new Date();
        const tomorrow = new Date();
        tomorrow.setDate(today.getDate() + 1);
        const dayAfterTomorrow = new Date();
        dayAfterTomorrow.setDate(today.getDate() + 2);
        const dayAfterAfterTomorrow = new Date();
        dayAfterAfterTomorrow.setDate(today.getDate() + 3);
        const dayAfterAfterAfterTomorrow = new Date();
        dayAfterAfterAfterTomorrow.setDate(today.getDate() + 4);
        const days = [tomorrow, dayAfterTomorrow, dayAfterAfterTomorrow, dayAfterAfterAfterTomorrow];

        days.forEach(day => {
            const dayData = data.list.filter(item => {
                const itemDate = new Date(item.dt_txt);
                return itemDate.getDate() === day.getDate();
            });

            if (dayData.length > 0) {
                const temps = dayData.map(item => item.main.temp);
                const tempAvg = temps.reduce((a, b) => a + b) / temps.length;

                const description = dayData[0].weather[0].description;

                const tempMin = Math.min(...dayData.map(item => item.main.temp_min));
                const tempMax = Math.max(...dayData.map(item => item.main.temp_max));

                forecastData.push({
                    date: day.toLocaleDateString('tr-TR', { weekday: 'long', day: 'numeric', month: 'long' }),
                    temp: Math.round(tempAvg),
                    description: description,
                    temp_min: Math.round(tempMin),
                    temp_max: Math.round(tempMax)
                });
            }
        });
    };

    const getSuggestions = (weather) => {
        const temp = weather.main.temp;
        const description = weather.weather[0].description.toLowerCase();

        if (description.includes('yağmur')) {
            suggestion = 'Bugün yağmur yağması bekleniyor, şemsiye almayı unutmayın.';
        } else if (description.includes('kar')) {
            suggestion = 'Bugün kar yağışı var, dikkatli yürümeye özen gösterin.';
        } else if (description.includes('açık')) {
            if (temp > 25) {
                suggestion = 'Bugün hava sıcak, hafif giysiler giymeyi tercih edebilirsiniz.';
            } else if (temp > 15) {
                suggestion = 'Bugün dışarıda güzel vakit geçirmek için harika bir gün!';
            } else {
                suggestion = 'Hava serin, bir ceket almayı unutmayın.';
            }
        } else {
            suggestion = 'Bugün hava kapalı, iç mekan aktivitelerini değerlendirebilirsiniz.';
        }
    };

    const handleSubmit = event => {
        event.preventDefault();
        getResult();
    };

    const handleInstallClick = async () => {
        deferredPrompt.prompt();
        const { outcome } = await deferredPrompt.userChoice;
        if (outcome === 'accepted') {
            console.log('Kullanıcı uygulamayı kurmayı kabul etti');
        } else {
            console.log('Kullanıcı uygulamayı kurmayı reddetti');
        }
        deferredPrompt = null;
        showInstallModal = false;
    };

    const closeModal = () => {
        showInstallModal = false;
    };

    const getIconPath = (description) => {
        if (description.includes('parçalı')) {
            return '/havaicon/parcalibulut.png';
        } else if (description.includes('açık')) {
            return '/havaicon/acik.png';
        } else if (description.includes('kapalı')) {
            return '/havaicon/bulutlu.png';
        } else if (description.includes('kar')) {
            return '/havaicon/karli.png';
        } else if (description.includes('yağmur')) {
            return '/havaicon/yagmurlu.png';
        } else {
            return '/havaicon/default.png';
        }
    };
</script>

{#if showInstallModal}
<div class="fixed inset-0 flex items-center justify-center z-50">
    <div class="bg-black opacity-50 absolute inset-0"></div>
    <div class="bg-white p-6 rounded-lg z-10 max-w-sm w-full text-center">
        <h2 class="text-xl font-bold mb-4">Uygulamayı İndir</h2>
        <p class="mb-4">Bu uygulamayı cihazınıza kurmak ister misiniz?</p>
        <div class="flex justify-between">
            <button
                on:click={handleInstallClick}
                class="bg-green-600 text-white py-2 px-4 rounded-lg hover:bg-green-500"
            >
                Evet
            </button>
            <button
                on:click={closeModal}
                class="bg-red-600 text-white py-2 px-4 rounded-lg hover:bg-red-500"
            >
                Hayır
            </button>
        </div>
    </div>
</div>
{/if}

<div
  class="app flex flex-col items-center text-center min-h-screen p-4 bg-cover bg-center"
  style="background-image: url('https://source.unsplash.com/featured/?weather');"
>
  <div class="header w-full max-w-lg py-8">
    <h1 class="text-4xl sm:text-5xl text-pink-700 font-bold flex items-center justify-center">
      <img src="/byk-logo.png" class="w-12 mr-2" alt="Logo" />
      BYK Hava Durumu
    </h1>
    <form
      on:submit={handleSubmit}
      class="w-full mt-6 flex flex-col sm:flex-row items-center justify-center"
    >
      <input
        type="text"
        bind:value={cityName}
        placeholder="Şehir Giriniz"
        class="flex-1 py-2 px-4 w-full border-b-2 border-pink-700 text-pink-700 text-center text-lg bg-transparent outline-none focus:border-pink-500"
      />
      <button
        type="submit"
        class="mt-4 sm:mt-0 sm:ml-4 px-6 py-2 bg-pink-700 text-white font-bold rounded-lg shadow-md hover:bg-pink-600 transition w-full sm:w-auto"
      >
        Ara
      </button>
    </form>
    {#if errorMessage}
    <div class="text-red-500 mt-4">{errorMessage}</div>
    {/if}
  </div>

  {#if isLoading}
  <div class="mt-8 text-pink-700 text-xl">Yükleniyor...  </div>
  {:else if weatherData}
  <div
    class="content w-full max-w-lg bg-white bg-opacity-70 backdrop-filter backdrop-blur-lg p-6 rounded-xl shadow-lg mt-8"
  >
    <div class="city text-3xl sm:text-4xl font-semibold mb-2">
      {weatherData.name}, {weatherData.sys.country}
    </div>
    <div class="temp text-5xl sm:text-6xl font-bold mb-4">
      {Math.round(weatherData.main.temp)}°C
    </div>
    <div class="desc text-lg sm:text-xl italic mb-2 capitalize text-gray-700">
      {weatherData.weather[0].description}
    </div>
    <img src={getIconPath(weatherData.weather[0].description)} alt="Weather Icon" class="w-24 h-24 mx-auto mb-4" />
    <div class="minmax text-sm sm:text-base text-gray-600">
      En Yüksek: {Math.round(weatherData.main.temp_max)}°C | En Düşük: {Math.round(weatherData.main.temp_min)}°C
    </div>
  </div>

  {#if forecastData.length > 0}
  <div class="forecast w-full max-w-lg mt-8">
    <div class="grid grid-cols-2 sm:grid-cols-3 gap-4">
      {#each forecastData as day}
      <div
        class="forecast-card bg-white bg-opacity-70 backdrop-filter backdrop-blur-lg p-4 rounded-xl shadow-md text-center"
      >
        <div class="date text-sm font-medium text-gray-700">{day.date}</div>
        <div class="temp text-2xl font-bold mt-2">{day.temp}°C</div>
        <img src={getIconPath(day.description)} alt="Forecast Icon" class="w-16 h-16 mx-auto mt-2" />
        <div class="description text-sm capitalize text-gray-600 mt-1">
          {day.description}
        </div>
        <div class="minmax text-xs text-gray-500 mt-1">
          {day.temp_max}°C / {day.temp_min}°C
        </div>
      </div>
      {/each}
    </div>
  </div>
  {/if}

  {#if suggestion}
  <div class="suggestion bg-yellow-100 p-4 rounded-lg mt-6 text-center mb-4">
    <h3 class="text-lg font-semibold">Günün Önerisi</h3>
    <p class="text-gray-700">{suggestion}</p>
  </div>
  {/if}

  {/if}
</div>
