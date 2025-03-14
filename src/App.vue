<template>
  <v-app>
    <v-container class="pa-6">
      <v-card class="pa-5">
        <v-card-title class="text-h5">YouTube Chinese Subtitle Analyzer</v-card-title>

        <!-- Fixed Categories -->
        <v-chip-group mandatory>
          <v-chip
            v-for="category in categories"
            :key="category"
            :class="{ 'font-weight-bold': selectedCategory === category }"
            @click="fetchCategoryVideos(category)"
          >
            {{ category }}
          </v-chip>
        </v-chip-group>

        <!-- Loading Spinner -->
        <v-progress-circular
          v-if="loadingVideos"
          indeterminate
          color="primary"
          class="mt-5"
        ></v-progress-circular>

        <!-- Video Carousel (3 videos per slide) -->
        <v-carousel
          v-if="!loadingVideos && videoSlides.length > 0"
          hide-delimiters
          cycle
          height="300"
          class="mt-5"
          interval="5000"
          show-arrows="hover"
        >
          <v-carousel-item
            v-for="(videoGroup, index) in videoSlides"
            :key="index"
          >
            <v-row>
              <v-col
                v-for="(video, i) in videoGroup"
                :key="i"
                cols="4"
              >
                <v-card @click="addUrl(video.url)" class="pa-3" elevation="4">
                  <v-img :src="video.thumbnail" height="200"></v-img>
                  <v-card-title class="text-subtitle-1">{{ video.title }}</v-card-title>
                  <v-card-subtitle>{{ video.duration }} | {{ video.views }} views</v-card-subtitle>
                </v-card>
              </v-col>
            </v-row>
          </v-carousel-item>
        </v-carousel>

        <v-divider class="my-4"></v-divider>

        <!-- Textarea for User Input -->
        <v-textarea
          v-model="urlsInput"
          label="Enter YouTube URLs (one per line)"
          outlined
          rows="3"
          class="mt-4"
        ></v-textarea>

        <!-- Analyze Button -->
        <v-btn
          color="primary"
          class="mt-3"
          @click="analyzeVideos"
          :loading="loading"
        >
          Analyze Subtitles
        </v-btn>

        <!-- Error Alert -->
        <v-alert
          v-if="error"
          type="error"
          class="mt-3"
          dismissible
        >
          {{ error }}
        </v-alert>

        <v-divider class="my-4"></v-divider>

        <!-- Common Words Table -->
        <v-data-table
          v-if="Array.isArray(totalWords) && totalWords.length > 0"
          :headers="headers"
          :items="totalWords"
          class="elevation-2"
        >
          <template v-slot:top>
            <v-toolbar flat>
              <v-toolbar-title>Common Words Across Videos</v-toolbar-title>
            </v-toolbar>
          </template>
          <template v-slot:item="{ item }">
            <tr>
              <td>{{ item.word }}</td>
              <td>{{ item.count }}</td>
            </tr>
          </template>
        </v-data-table>

        <!-- Results Per Video -->
        <v-card
          v-for="(words, url) in videoWordCounts"
          :key="url"
          class="mt-5 pa-3"
        >
          <v-card-title class="text-subtitle-1">Results for: {{ url }}</v-card-title>
          <v-data-table
            v-if="Array.isArray(words) && words.length > 0"
            :headers="headers"
            :items="words"
            dense
          >
            <template v-slot:item="{ item }">
              <tr>
                <td>{{ item.word }}</td>
                <td>{{ item.count }}</td>
              </tr>
            </template>
          </v-data-table>
        </v-card>
      </v-card>
    </v-container>
  </v-app>
</template>

<script>
import axios from "axios";

export default {
  name: "App",
  data() {
    return {
      urlsInput: "",
      videoResults: [], // Stores video metadata
      videoSlides: [], // Groups of 3 videos per carousel slide
      videoWordCounts: {}, // Stores subtitle analysis per video
      totalWords: [], // Stores common words across all videos
      error: null,
      loading: false,
      loadingVideos: false, // Loading state for fetching videos
      selectedCategory: "新闻", // Default to 新闻 (News)
      categories: ["新闻", "音乐", "科技", "电影", "游戏"],
      headers: [
        { text: "Word", value: "word" },
        { text: "Frequency", value: "count" }
      ]
    };
  },
  mounted() {
    this.fetchCategoryVideos(this.selectedCategory);
  },
  methods: {
    // Fetch category videos
    async fetchCategoryVideos(category) {
      this.selectedCategory = category;
      this.videoResults = []; // Reset videos
      this.videoSlides = []; // Reset slides
      this.loadingVideos = true; // Show loading spinner

      try {
        const response = await axios.get("http://localhost:5001/search", {
          params: { category }
        });

        console.log("API Response:", response.data); // Debugging log

        if (!response.data.results || !Array.isArray(response.data.results)) {
          throw new Error("Invalid API response format");
        }

        this.videoResults = response.data.results;

        // Ensure at least 3 videos
        while (this.videoResults.length < 3) {
          this.videoResults.push({
            url: "#",
            thumbnail: "https://via.placeholder.com/300x200",
            title: "Placeholder Video",
            duration: "0:00",
            views: "N/A"
          });
        }

        // Convert videoResults into slides of 3 videos each
        this.videoSlides = this.chunkArray(this.videoResults, 3);
      } catch (err) {
        console.error("Error fetching videos:", err); // Debug log
        this.error = "Error fetching category videos.";
        this.videoResults = []; // Avoid null issues
        this.videoSlides = [];
      }

      this.loadingVideos = false; // Hide loading spinner
    },

    // Add video URL to textarea
    addUrl(url) {
      if (!this.urlsInput.includes(url)) {
        this.urlsInput += (this.urlsInput ? "\n" : "") + url;
      }
    },

    // Analyze Videos
    async analyzeVideos() {
      this.error = null;
      this.videoWordCounts = {}; // Reset per-video word counts
      this.totalWords = []; // Reset total word analysis
      const urls = this.urlsInput.split("\n").map(url => url.trim()).filter(url => url);

      if (urls.length === 0) {
        this.error = "Please enter at least one YouTube URL.";
        return;
      }

      this.loading = true;
      try {
        const response = await axios.post("http://localhost:5001/analyze", { urls });

        this.totalWords = response.data.total_common_words.map(([word, count]) => ({ word, count }));

        // Store per-video analysis separately
        this.videoWordCounts = {};
        for (const url in response.data.video_results) {
          if (Array.isArray(response.data.video_results[url])) {
            this.videoWordCounts[url] = response.data.video_results[url].map(([word, count]) => ({ word, count }));
          }
        }
      } catch (err) {
        this.error = "Invalid Videos";
      }
      this.loading = false;
    },

    // Helper function to split videos into groups of 3
    chunkArray(array, size) {
      const chunkedArr = [];
      for (let i = 0; i < array.length; i += size) {
        chunkedArr.push(array.slice(i, i + size));
      }
      return chunkedArr;
    }
  }
};
</script>

<style>
/* Custom styles if needed */
</style>
