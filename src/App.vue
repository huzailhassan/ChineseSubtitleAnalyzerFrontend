<template>
  <v-app>
    <v-container class="pa-6">
      <v-card class="pa-5">
        <v-card-title class="text-h5">YouTube Chinese Subtitle Analyzer</v-card-title>

        <!-- Textarea for user input -->
        <v-textarea
          v-model="urlsInput"
          label="Enter YouTube URLs (one per line)"
          outlined
          rows="3"
          class="mt-4"
        ></v-textarea>

        <!-- Analyze button -->
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

        <!-- Overall Common Words Table -->
        <v-data-table
          v-if="totalWords.length > 0"
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
          v-for="(words, url) in videoResults"
          :key="url"
          class="mt-5 pa-3"
        >
          <v-card-title class="text-subtitle-1">Results for: {{ url }}</v-card-title>
          <v-data-table
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
      videoResults: {},
      totalWords: [],
      loading: false,
      error: null,
      headers: [
        { text: "Word", value: "word" },
        { text: "Frequency", value: "count" }
      ]
    };
  },
  methods: {
    async analyzeVideos() {
      this.error = null;
      this.videoResults = {};
      this.totalWords = [];
      const urls = this.urlsInput
        .split("\n")
        .map(url => url.trim())
        .filter(url => url);

      if (urls.length === 0) {
        this.error = "Please enter at least one YouTube URL.";
        return;
      }

      this.loading = true;
      try {
        // Make sure this URL matches your Flask server address/port
        const response = await axios.post("http://127.0.0.1:5001/analyze", { urls });
        
        this.videoResults = response.data.video_results;

        // Convert total_common_words to an array of objects [{word, count}]
        this.totalWords = response.data.total_common_words.map(([word, count]) => ({ word, count }));
        
        // Convert each video's result to [{word, count}]
        for (const url in this.videoResults) {
          if (Array.isArray(this.videoResults[url])) {
            this.videoResults[url] = this.videoResults[url].map(([word, count]) => ({
              word,
              count
            }));
          }
        }
      } catch (err) {
        this.error = "Failed to analyze videos. Check your Flask server or video URLs.";
      }
      this.loading = false;
    }
  }
};
</script>

<style>
/* You can add global styles here, or do nothing */
</style>
