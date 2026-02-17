<template>
  <div class="weather" v-if="weatherData.adCode.city && weatherData.weather.weather">
    <span>{{ weatherData.adCode.city }}&nbsp;</span>
    <span>{{ weatherData.weather.weather }}&nbsp;</span>
    <span>{{ weatherData.weather.temperature }}℃</span>
    <span class="sm-hidden">
      &nbsp;{{
        weatherData.weather.winddirection?.endsWith("风")
          ? weatherData.weather.winddirection
          : weatherData.weather.winddirection + "风"
      }}&nbsp;
    </span>
    <span class="sm-hidden">{{ weatherData.weather.windpower }}&nbsp;级</span>
  </div>
  <div class="weather" v-else-if="isLoading">
    <span>天气信息获取中...</span>
  </div>
  <div class="weather" v-else>
    <span>天气数据获取失败</span>
  </div>
</template>

<script setup>
import { getAdcode, getWeather, getOtherWeather, getRegeo } from "@/api";
import { Error } from "@icon-park/vue-next";
import { reactive, ref, onMounted, h } from "vue";

// 高德开发者 Key
const mainKey = import.meta.env.VITE_WEATHER_KEY;
// 备用数据源的 高德 Key
const backupKey = "03c558fd7bc4fd3829dd2c1d53afbb9f";

// 控制加载状态的变量
const isLoading = ref(true);

// 天气数据
const weatherData = reactive({
  adCode: {
    city: null, // 城市
    adcode: null, // 城市编码
  },
  weather: {
    weather: null, // 天气现象
    temperature: null, // 实时气温
    winddirection: null, // 风向描述
    windpower: null, // 风力级别
  },
});

// 取出天气平均值
const getTemperature = (min, max) => {
  try {
    // 计算平均值并四舍五入
    const average = (Number(min) + Number(max)) / 2;
    return Math.round(average);
  } catch (error) {
    console.error("计算温度出现错误：", error);
    return "NaN";
  }
};

// 获取设备的真实经纬度定位
const getPosition = () => {
  return new Promise((resolve, reject) => {
    if ("geolocation" in navigator) {
      navigator.geolocation.getCurrentPosition(
        (position) => {
          resolve(`${position.coords.longitude},${position.coords.latitude}`);
        },
        (error) => {
          reject(error);
        },
        { enableHighAccuracy: true, timeout: 5000, maximumAge: 0 }
      );
    } else {
      reject(new Error("浏览器不支持地理定位"));
    }
  });
};

// 获取天气数据
const getWeatherData = async () => {
  // 请求开始，开启加载状态
  isLoading.value = true;
  
  try {
    let adcode = "";
    let city = "";
    // 优先使用用户在环境变量里配置的key，如果没有则用提供的备用key
    const finalKey = mainKey || backupKey;

    // 1. 尝试动态获取当前定位的行政区划代码 (真实定位而非代理IP)
    try {
      const location = await getPosition();
      const regeoRes = await getRegeo(finalKey, location);
      if (regeoRes.status === "1" && regeoRes.regeocode) {
        adcode = regeoRes.regeocode.addressComponent.adcode;
        city = regeoRes.regeocode.addressComponent.city || regeoRes.regeocode.addressComponent.province;
      }
    } catch (geoErr) {
      console.warn("真实定位获取失败，降级使用 IP 定位", geoErr);
      // 降级：如果浏览器拒绝定位或在HTTP环境下，使用原有的 IP 定位
      const adCodeRes = await getAdcode(finalKey);
      if (adCodeRes.infocode === "10000" && adCodeRes.adcode) {
        adcode = adCodeRes.adcode;
        city = adCodeRes.city;
      }
    }

    if (!adcode) {
      throw new Error("地区查询失败");
    }

    weatherData.adCode = {
      city: city,
      adcode: adcode,
    };

    // 2. 调用高德天气 API
    if (mainKey) {
      // 保持原有逻辑：如果用户配置了独立的 mainKey，使用实时天气 (extensions=base)
      const result = await getWeather(mainKey, adcode, "base");
      weatherData.weather = {
        weather: result.lives[0].weather,
        temperature: result.lives[0].temperature,
        winddirection: result.lives[0].winddirection,
        windpower: result.lives[0].windpower,
      };
    } else {
      // 备用数据源逻辑：使用 extensions=all 预报接口并计算数值
      const result = await getWeather(backupKey, adcode, "all");
      if (result.status === "1" && result.forecasts.length > 0) {
        const cast = result.forecasts[0].casts[0]; // 获取当天的预报数组
        weatherData.weather = {
          weather: cast.dayweather,
          temperature: getTemperature(cast.nighttemp, cast.daytemp), // 用预报的高低温取平均作为实时温度
          winddirection: cast.daywind,
          windpower: cast.daypower,
        };
      }
    }
  } catch (error) {
    console.warn("高德天气获取失败，尝试兜底教书先生接口:" + error);
    // 3. 终极兜底：教书先生天气接口
    try {
      const result = await getOtherWeather();
      const data = result.result;
      weatherData.adCode = {
        city: data.city.City || "未知地区",
      };
      weatherData.weather = {
        weather: data.condition.day_weather,
        temperature: getTemperature(data.condition.min_degree, data.condition.max_degree),
        winddirection: data.condition.day_wind_direction,
        windpower: data.condition.day_wind_power,
      };
    } catch (err) {
      console.error("天气信息获取完全失败:" + err);
      onError("天气信息获取失败");
    }
  } finally {
    // 无论最终成功还是彻底失败，请求结束时关闭加载状态
    isLoading.value = false;
  }
};

// 报错信息
const onError = (message) => {
  ElMessage({
    message,
    icon: h(Error, {
      theme: "filled",
      fill: "#efefef",
    }),
  });
  console.error(message);
};

onMounted(() => {
  // 调用获取天气
  getWeatherData();
});
</script>