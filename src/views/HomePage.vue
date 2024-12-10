<template>
  <ion-page>
    <ion-header :translucent="true">
      <ion-toolbar>
        <ion-title>Home</ion-title>
      </ion-toolbar>
    </ion-header>

    <ion-content :fullscreen="true">
      <!-- Refresh functionality -->
      <ion-refresher slot="fixed" @ionRefresh="handleRefresh($event)">
        <ion-refresher-content></ion-refresher-content>
      </ion-refresher>

      <!-- Todo List -->
      <ion-list class="scrollable-container">
        <ion-item v-for="todo in todos" :key="todo.id">
          <ion-card>
            <ion-card-header>
              <ion-card-subtitle class="limited-text">{{
                todo.description
              }}</ion-card-subtitle>
              <ion-card-title class="ion-text-wrap limited-text">{{
                todo.title
              }}</ion-card-title>
            </ion-card-header>

            <ion-card-content class="card-actions">
              <ion-badge>{{ getRelativeTime(todo.updatedAt) }}</ion-badge>
              <div>
                <ion-button
                  size="small"
                  color="primary"
                  @click="handleEdit(todo)"
                >
                  Edit
                </ion-button>
                <ion-button
                  size="small"
                  color="danger"
                  @click="todo.id && handleDelete(todo.id)"
                >
                  Delete
                </ion-button>
              </div>
            </ion-card-content>
          </ion-card>
        </ion-item>
        <ion-item v-if="todos.length === 0" class="ion-text-center">
          <ion-label>No todos available</ion-label>
        </ion-item>
      </ion-list>

      <!-- Add Todo Button -->
      <ion-fab vertical="bottom" horizontal="end" slot="fixed">
        <ion-fab-button @click="isOpen = true">
          <ion-icon :icon="add" size="large"></ion-icon>
        </ion-fab-button>
      </ion-fab>
      <InputModal
        v-model:isOpen="isOpen"
        v-model:editingId="editingId"
        :todo="todo"
        @submit="handleSubmit"
      />
    </ion-content>
  </ion-page>
</template>

<script setup lang="ts">
import {
  IonContent,
  IonHeader,
  IonPage,
  IonTitle,
  IonToolbar,
  IonItem,
  IonCard,
  IonCardHeader,
  IonCardTitle,
  IonCardSubtitle,
  IonCardContent,
  IonBadge,
  IonButton,
  IonFab,
  IonFabButton,
  IonIcon,
  IonRefresher,
  IonRefresherContent,
  toastController,
} from "@ionic/vue";
import { add } from "ionicons/icons";
import InputModal from "@/components/InputModal.vue";
import { onMounted, ref, onUnmounted } from "vue";
import { firestoreService, type Todo } from "@/utils/firestore";
import { formatDistanceToNow } from "date-fns";

// State variables
const isOpen = ref(false);
const editingId = ref<string | null>(null);
const todos = ref<Todo[]>([]);
const todo = ref<Omit<Todo, "id" | "createdAt" | "updatedAt" | "status">>({
  title: "",
  description: "",
});
const timeUpdateTrigger = ref(0);

// Toast notifications
const showToast = async (message: string, color: string = "success") => {
  const toast = await toastController.create({
    message,
    duration: 2000,
    color,
    position: "top",
  });
  await toast.present();
};

// Relative time for todo updates
const getRelativeTime = (date: any) => {
  timeUpdateTrigger.value;
  try {
    const jsDate = date?.toDate ? date.toDate() : new Date(date);
    return formatDistanceToNow(jsDate, { addSuffix: true });
  } catch (error) {
    return "Invalid date";
  }
};

// Handle refresh
const handleRefresh = async (event: any) => {
  try {
    await loadTodos();
  } catch (error) {
    console.error("Error refreshing:", error);
  } finally {
    event.target.complete();
  }
};

// Handle submit (add/edit)
const handleSubmit = async (
  todo: Omit<Todo, "id" | "createdAt" | "updatedAt" | "status">
) => {
  if (!todo.title) {
    await showToast("Title is required", "warning");
    return;
  }
  try {
    if (editingId.value) {
      await firestoreService.updateTodo(editingId.value, todo as Todo);
      await showToast("Todo updated successfully");
    } else {
      await firestoreService.addTodo(todo as Todo);
      await showToast("Todo added successfully");
    }
    loadTodos();
  } catch (error) {
    await showToast("An error occurred", "danger");
    console.error(error);
  } finally {
    editingId.value = null;
  }
};

// Load todos
const loadTodos = async () => {
  try {
    todos.value = await firestoreService.getTodos();
  } catch (error) {
    console.error(error);
  }
};

// Handle edit
const handleEdit = async (editTodo: Todo) => {
  editingId.value = editTodo.id!;
  todo.value = {
    title: editTodo.title,
    description: editTodo.description,
  };
  isOpen.value = true;
};

// Handle delete
const handleDelete = async (id: string) => {
  try {
    await firestoreService.deleteTodo(id); // Pastikan ada metode ini di firestoreService
    await showToast("Todo deleted successfully");
    loadTodos(); // Refresh daftar setelah penghapusan
  } catch (error) {
    await showToast("An error occurred while deleting", "danger");
    console.error(error);
  }
};

// Lifecycle hooks
let intervalId: ReturnType<typeof setInterval>;

onMounted(() => {
  loadTodos();
  intervalId = setInterval(() => {
    timeUpdateTrigger.value++;
  }, 60000);
});

onUnmounted(() => {
  clearInterval(intervalId);
});
</script>

<style scoped>
.scrollable-container {
  max-height: 80vh;
  overflow-y: auto;
  padding: 0 1rem; /* Memberikan padding pada container */
}

.limited-text {
  overflow: hidden;
  text-overflow: ellipsis;
  display: block; /* Pastikan ini blok untuk wrapping otomatis */
  white-space: normal; /* Mengatur agar teks tidak terpotong */
}

ion-card {
  margin: 1rem auto; /* Margin atas dan bawah, simetris kanan kiri */
  width: calc(100% - 2rem); /* Pastikan kartu memiliki margin kanan dan kiri */
  max-width: 600px; /* Membatasi lebar maksimal kartu */
}

ion-card-title.limited-text {
  -webkit-line-clamp: unset; /* Membebaskan batasan baris */
}

ion-card-subtitle.limited-text {
  -webkit-line-clamp: unset; /* Membebaskan batasan baris */
}

.card-actions {
  display: flex;
  justify-content: space-between; /* Menyelaraskan elemen di kanan dan kiri */
  align-items: center; /* Menyelaraskan secara vertikal */
  gap: 1rem; /* Jarak antar elemen */
}

ion-badge {
  font-size: 0.85rem; /* Menyesuaikan ukuran badge agar proporsional */
}

ion-button {
  --padding-start: 0.5rem;
  --padding-end: 0.5rem;
  --border-radius: 0.25rem; /* Membuat tombol lebih kecil dan ramping */
}
</style>
