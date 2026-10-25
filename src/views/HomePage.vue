<template>
  <ion-page>
    <ion-header class="ion-no-border">
      <ion-toolbar class="custom-toolbar">
        <ion-title class="app-title">
          <span class="title-icon">🌤️</span> Prakiraan Cuaca Jakarta
        </ion-title>

        <ion-buttons slot="end">
          <ion-button @click="toggleTheme" fill="clear">
            {{ isDarkMode ? ' ☀️ Light' : ' 🌙 Dark' }}
          </ion-button>
        </ion-buttons>
      </ion-toolbar>
    </ion-header>

    <ion-content :class="[isDarkMode ? 'dark-container' : 'light-container', 'ion-padding']">
      <div class="content-wrapper">
        
        <div class="identity-banner">
          <div class="user-info">
            <span class="badge-student name-badge">👤 Anggi Tripambudi</span>
            <span class="badge-student nim-badge">🎓 NIM: 054183654</span>
          </div>
          <div class="location-badge">
            <span>📍 DKI Jakarta (-6.2, 106.8)</span>
          </div>
        </div>

        <div class="stats-grid" v-if="!loading && !errorMsg">
          <div class="stat-card">
            <div class="stat-title">Suhu Rata-rata</div>
            <div class="stat-value text-blue">{{ stats.avg }} °C</div>
          </div>
          <div class="stat-card">
            <div class="stat-title">Suhu Tertinggi</div>
            <div class="stat-value text-amber">{{ stats.max }} °C</div>
          </div>
          <div class="stat-card">
            <div class="stat-title">Suhu Terendah</div>
            <div class="stat-value text-cyan">{{ stats.min }} °C</div>
          </div>
        </div>

        <div class="card-glass">
          <div class="table-header-title">
            <h3>Prakiraan Suhu Per Jam</h3>
            <span class="sub-title">Data real-time via REST API Open-Meteo</span>
          </div>

          <div class="table-responsive">
            <table class="modern-table">
              <thead>
                <tr>
                  <th class="text-center">No</th>
                  <th>Hari</th>
                  <th>Tanggal</th>
                  <th>Jam</th>
                  <th class="text-center">Kondisi</th>
                  <th class="text-right">Suhu Udara</th>
                </tr>
              </thead>
              <tbody>
                <tr v-if="loading">
                  <td colspan="6" class="state-cell">
                    <p>Mengambil data dari server API...</p>
                  </td>
                </tr>
                <tr v-else-if="errorMsg">
                  <td colspan="6" class="state-cell text-error">
                    ⚠️ {{ errorMsg }}
                  </td>
                </tr>
                <tr v-else v-for="(item, index) in dataTampil" :key="index" class="table-row">
                  <td class="text-center row-num">{{ index + 1 }}</td>
                  <td class="font-semibold">{{ item.hari }}</td>
                  <td class="text-muted">{{ item.tanggal }}</td>
                  <td>
                    <span class="time-chip">{{ item.jam }}</span>
                  </td>
                  <td class="text-center">
                    <span :class="['weather-tag', item.kondisi.kelas]">
                      {{ item.kondisi.label }}
                    </span>
                  </td>
                  <td class="text-right">
                    <span class="temp-badge">{{ item.suhu }} °C</span>
                  </td>
                </tr>
              </tbody>
            </table>
          </div>

          <div class="table-footer-controls">
            <div class="filter-group">
              <label for="rowLimit">Tampilkan Baris Data:</label>
              <ion-item lines="none" class="custom-select-item">
                <ion-select 
                  v-model="rowLimit" 
                  @ionChange="updateTampilan" 
                  interface="popover"
                  class="custom-select"
                >
                  <ion-select-option value="10">10 Baris Pertama</ion-select-option>
                  <ion-select-option value="20">20 Baris Pertama</ion-select-option>
                  <ion-select-option value="30">30 Baris Pertama</ion-select-option>
                  <ion-select-option value="all">Semua Data API</ion-select-option>
                </ion-select>
              </ion-item>
            </div>
          </div>
        </div>

      </div>
    </ion-content>
  </ion-page>
</template>

<script lang="ts">
import { 
  IonPage, IonHeader, IonToolbar, IonTitle, IonContent, 
  IonItem, IonSelect, IonSelectOption 
} from '@ionic/vue';
import { defineComponent } from 'vue';
import EndPointAccess from '@/services/EndPointAccess';

interface WeatherItem {
  hari: string;
  tanggal: string;
  jam: string;
  suhu: number;
  kondisi: { label: string; kelas: string };
}

export default defineComponent({
  name: 'HomePage',
  components: {
    IonPage, IonHeader, IonToolbar, IonTitle, IonContent,
    IonItem, IonSelect, IonSelectOption
  },
  data() {
    return {
      isDarkMode: true,
      rawTimes: [] as string[],
      rawTemps: [] as number[],
      dataTampil: [] as WeatherItem[],
      rowLimit: '10',
      loading: true,
      errorMsg: '',
      namaHari: ['Minggu', 'Senin', 'Selasa', 'Rabu', 'Kamis', 'Jumat', 'Sabtu'],
      stats: {
        avg: '0',
        max: '0',
        min: '0'
      }
    };
  },
  mounted() {
    this.ambilDataCuaca();
  },
  methods: {
    toggleTheme() {
      this.isDarkMode = !this.isDarkMode;
    },

    pindahHalaman(rute: string) {
      this.$router.push(rute);
    },
    
    async ambilDataCuaca() {
      const url = 'https://api.open-meteo.com/v1/forecast?latitude=-6.2&longitude=106.8&hourly=temperature_2m';
      const service = new EndPointAccess(url);

      try {
        const response = await service.getRes();
        this.rawTimes = response.data.hourly.time;
        this.rawTemps = response.data.hourly.temperature_2m;
        
        if (this.rawTemps.length > 0) {
          const sum = this.rawTemps.reduce((a, b) => a + b, 0);
          this.stats.avg = (sum / this.rawTemps.length).toFixed(1);
          this.stats.max = Math.max(...this.rawTemps).toString();
          this.stats.min = Math.min(...this.rawTemps).toString();
        }

        this.loading = false;
        this.updateTampilan();
      } catch (error: any) {
        this.loading = false;
        this.errorMsg = 'Gagal mengambil data dari API: ' + error.message;
      }
    },
    getKondisiCuaca(suhu: number) {
      if (suhu >= 30) return { label: '☀️ Cerah', kelas: 'cerah' };
      if (suhu >= 27) return { label: '⛅ Berawan', kelas: 'berawan' };
      if (suhu >= 24) return { label: '☁️ Mendung', kelas: 'mendung' };
      return { label: '🌧️ Hujan / Sejuk', kelas: 'hujan' };
    },
    updateTampilan() {
      const total = this.rowLimit === 'all' 
        ? this.rawTimes.length 
        : Math.min(parseInt(this.rowLimit), this.rawTimes.length);

      const hasil: WeatherItem[] = [];

      for (let i = 0; i < total; i++) {
        const [tanggal, jam] = this.rawTimes[i].split('T');
        const d = new Date(this.rawTimes[i]);
        const hari = this.namaHari[d.getDay()];
        const suhu = this.rawTemps[i];

        hasil.push({
          hari,
          tanggal,
          jam,
          suhu,
          kondisi: this.getKondisiCuaca(suhu)
        });
      }

      this.dataTampil = hasil;
    }
  }
});
</script>

<style scoped src="./HomePage.css"></style>