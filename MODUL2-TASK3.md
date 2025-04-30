## Task 4 - Cella's Manhwa

Membantu Cella untuk mengumpulkan informasi dan foto dari berbagai manhwa favoritnya dengan skrip otomatis.

Daftar Manhwa :

|    No     |      Manhwa      |
| :--------: | :------------: |
| 1 | Mistaken as the Monster Duke's Wife |
| 2 | The Villainess Lives Again |
| 3 | No, I Only Charmed the Princess! |
| 4 | Darling, Why Can't We Divorce? |

Library yang digunakan :

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <curl/curl.h>
#include <jansson.h>
#include <ctype.h>
#include <pthread.h>
#include <dirent.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <unistd.h>  
#include <wait.h> 
```


### **a. Summoning the Manhwa Stats**

Mengambil data detail dari manhwa menggunakan API Jikan

- Judul
- Status
- Tanggal rilis
- Genre
- Tema
- Author

Lalu disimpan dalam file .txt dengan format nama file tanpa karakter khusus dan spasi diganti dengan underscore. File disimpan dalam folder `Manhwa`.


### **b. Seal the Scrolls**

Setiap file .txt kemudian di-zip dan disimpan ke dalam folder `Archive`. Format nama file zip adalah inisial judul manhwa tanpa character.

**Code a & b :**
```
struct string {
    char *ptr;
    size_t len;
};

void init_string(struct string *s) {
    s->len = 0;
    s->ptr = malloc(1);
    if (s->ptr == NULL) {
        fprintf(stderr, "Out of memory!\n");
        exit(EXIT_FAILURE);
    }
    s->ptr[0] = '\0';
}

size_t writefunc(void *ptr, size_t size, size_t nmemb, struct string *s) {
    size_t new_len = s->len + size * nmemb;
    s->ptr = realloc(s->ptr, new_len + 1);
    if (s->ptr == NULL) {
        fprintf(stderr, "realloc() failed\n");
        exit(EXIT_FAILURE);
    }
    memcpy(s->ptr + s->len, ptr, size * nmemb);
    s->ptr[new_len] = '\0';
    s->len = new_len;

    return size * nmemb;
}

void make_directory(const char *dir_path) {
    pid_t pid = fork();
    if (pid == 0) {
        execlp("mkdir", "mkdir", "-p", dir_path, (char *)NULL);
        perror("execlp failed");
        exit(EXIT_FAILURE);
    } else if (pid < 0) {
        perror("fork failed");
        exit(EXIT_FAILURE);
    } else {
        wait(NULL);  
    }
}

char *filename(const char *title) {
    char *filename = malloc(strlen(title) * 2);
    int j = 0;
    for (int i = 0; title[i]; i++) {
        if (isalnum(title[i])) {
            filename[j++] = title[i];
        } else if (isspace(title[i])) {
            filename[j++] = '_';
        }
    }
    filename[j] = '\0';
    return filename;
}

char *initial_filename(const char *filename) {
    char *initial = malloc(16);
    int j = 0;
    if (isalpha(filename[0])) {
        initial[j++] = toupper(filename[0]);
    }
    for (int i = 1; filename[i]; i++) {
        if (filename[i - 1] == '_' && isalpha(filename[i])) {
            initial[j++] = toupper(filename[i]);
        }
    }
    initial[j] = '\0';
    return initial;
}

void txt_zip(const char *filename, const char *content) {
    make_directory("Manhwa"); 
    char path[256];
    snprintf(path, sizeof(path), "Manhwa/%s.txt", filename);

    FILE *f = fopen(path, "w");
    if (!f) {
        perror("fopen");
        return;
    }
    fprintf(f, "%s", content);
    fclose(f);

    make_directory("Archive"); 
    char *zip_name = initial_filename(filename);
    char zip_path[256];
    snprintf(zip_path, sizeof(zip_path), "Archive/%s.zip", zip_name);

    pid_t pid = fork();
    if (pid == 0) {
        execlp("zip", "zip", "-j", zip_path, path, (char *)NULL);
        perror("execlp failed");
        exit(EXIT_FAILURE);
    } else if (pid < 0) {
        perror("fork failed");
        exit(EXIT_FAILURE);
    } else {
        wait(NULL);  
    }
    printf("%s.zip is completed!\n", zip_name);
    free(zip_name);
}

void manhwa_info(int id, struct Heroine *h) {
    CURL *curl;
    CURLcode res;
    struct string s;
    char url[128];

    sprintf(url, "https://api.jikan.moe/v4/manga/%d", id);

    curl = curl_easy_init();
    if (curl) {
        init_string(&s);
        curl_easy_setopt(curl, CURLOPT_URL, url);
        curl_easy_setopt(curl, CURLOPT_WRITEFUNCTION, writefunc);
        curl_easy_setopt(curl, CURLOPT_WRITEDATA, &s);

        res = curl_easy_perform(curl);
        if (res != CURLE_OK) {
            fprintf(stderr, "curl_easy_perform() failed: %s\n", curl_easy_strerror(res));
        } else {
            json_error_t error;
            json_t *root = json_loads(s.ptr, 0, &error);
            if (root) {
                json_t *data = json_object_get(root, "data");
                const char *title = json_string_value(json_object_get(data, "title_english"));
                const char *status = json_string_value(json_object_get(data, "status"));
                const char *release_raw = json_string_value(json_object_get(json_object_get(data, "published"), "from"));

                char release[11];
                strncpy(release, release_raw, 10);
                release[10] = '\0';

                json_t *genres = json_object_get(data, "genres");
                char genre_list[256] = "";
                for (size_t i = 0; i < json_array_size(genres); i++) {
                    json_t *g = json_array_get(genres, i);
                    strcat(genre_list, json_string_value(json_object_get(g, "name")));
                    if (i != json_array_size(genres) - 1) strcat(genre_list, ", ");
                }

                json_t *themes = json_object_get(data, "themes");
                char theme_list[256] = "";
                for (size_t i = 0; i < json_array_size(themes); i++) {
                    json_t *t = json_array_get(themes, i);
                    strcat(theme_list, json_string_value(json_object_get(t, "name")));
                    if (i != json_array_size(themes) - 1) strcat(theme_list, ", ");
                }

                json_t *authors = json_object_get(data, "authors");
                char author_list[256] = "";
                for (size_t i = 0; i < json_array_size(authors); i++) {
                    json_t *a = json_array_get(authors, i);
                    strcat(author_list, json_string_value(json_object_get(a, "name")));
                    if (i != json_array_size(authors) - 1) strcat(author_list, ", ");
                }

                h->manhwa_filename = strdup(filename(title));

                char content[1024];
                snprintf(content, sizeof(content),
                         "Title: %s\nStatus: %s\nRelease: %s\nGenre: %s\nTheme: %s\nAuthor: %s\n",
                         title, status, release, genre_list, theme_list, author_list);

                txt_zip(h->manhwa_filename, content);
                json_decref(root);
            } else {
                fprintf(stderr, "JSON parse error: %s\n", error.text);
            }
        }
        curl_easy_cleanup(curl);
        free(s.ptr);
    }
}

int main() {
    int id[] = {168827, 147205, 169731, 175521}; 

    struct Heroine heroines[] = {
        {"Lia", 3, "https://i.pinimg.com/736x/11/d9/ad/11d9ad85a47892f5fd979a0209162377.jpg"},
        {"Artezia", 6, "https://i.pinimg.com/736x/cf/ab/a7/cfab73765912c97bc16865df4a9b3455.jpg"},
        {"Adelia", 4, "https://i.pinimg.com/736x/fd/26/a7/fd26a75cbc439e66ed6b55bbd5c904f2.jpg"},
        {"Ophelia", 10, "https://i.pinimg.com/736x/ba/a7/c7/baa7c7b496cc61848d17b471a0000cbd.jpg"}
    };
    
    for (int i = 0; i < 4; i++) {
        manhwa_info(id[i], &heroines[i]);
    }

    pthread_t threads[4];

    for (int i = 0; i < 4; i++) {
        pthread_create(&threads[i], NULL, download_images, &heroines[i]);
        pthread_join(threads[i], NULL);
    }

    cleanup_images();
}
```

**Output :**

Terminal :

![Screenshot from 2025-04-30 14-49-50](https://github.com/user-attachments/assets/f4a1cb05-939b-4472-8c11-b6983e44686f)

a. Summoning the Manhwa Stats

Folder :

![Screenshot from 2025-04-30 14-55-14](https://github.com/user-attachments/assets/f6297084-47c6-4889-a56e-a604deb2c0d2)

Isi folder :

![Screenshot from 2025-04-30 14-55-24](https://github.com/user-attachments/assets/0edf2f6d-bb41-4e0d-a722-761206854b71)

Isi txt (Mistaken_as_the_Monster_Dukes_Wife.txt):

![Screenshot from 2025-04-30 15-04-01](https://github.com/user-attachments/assets/83c0c095-767c-4deb-9d0b-ce9c051c319c)

Isi txt (The_Villainess_Lives_Again.txt):

![Screenshot from 2025-04-30 15-30-44](https://github.com/user-attachments/assets/510ba393-d454-4a3e-891a-c6e578ee67eb)

Isi txt (No_I_Only_Charmed_the_Princess.txt):

![Screenshot from 2025-04-30 15-30-47](https://github.com/user-attachments/assets/7044ced6-9fc1-45a7-82e8-f131a4ea709b)

Isi txt (Darling_Why_Cant_We_Divorce.txt):

![Screenshot from 2025-04-30 15-30-49](https://github.com/user-attachments/assets/7a237be8-3d62-4b6f-9202-f646be0c0944)


b. Seal the Scrolls

Folder :

![Screenshot from 2025-04-30 14-55-06](https://github.com/user-attachments/assets/840d266d-0206-4f89-b166-fe840d7ba1f2)

Isi folder :

![Screenshot from 2025-04-30 14-55-39](https://github.com/user-attachments/assets/0d649dcd-69de-453b-aefa-49854fff9228)


**Penjelasan :**

Function :

`struct string` :
- Menyimpan semua konten API melalui `curl_easy_perform` (dari function `manhwa_info`) sebagai string dinamis.

`void init_string(struct string *s)` :
- Inisialisasi struct string jadi string kosong dinamis.
- `malloc(1)` nyiapin 1 byte awal.
- Di-terminate dengan '\0' supaya menjadi string valid.

`size_t writefunc(void *ptr, size_t size, size_t nmemb, struct string *s)` :
- Fungsi callback supaya libcurl dapat menaruh data ke memori.
- Realloc ukuran baru `(s->len + size * nmemb)`. Salin data baru ke `s->ptr`. Update `s->len`.

`void make_directory(const char *dir_path)` :
- Membuat folder dengan fork + execlp.

`char *filename(const char *title)` :
- Mengonversi judul manhwa jadi nama file valid. Nama file tanpa karakter khusus dan spasi diganti dengan underscore.

`char *initial_filename(const char *filename)` :
- Mengambil huruf pertama dan setiap huruf setelah `_` untuk nama zip.

`void txt_zip(const char *filename, const char *content)` :
- Membuat direktori `Manhwa` dan `Archive`
- Menyimpan informasi manhwa ke dalam txt.
- Membuat file .zip dari file .txt.

`void manhwa_info(int id, struct Heroine *h)` :
- Mengambil informasi manhwa via Jikan API berdasarkan id dari array `id` di `main`.
- Menggabungkan informasi ke dalam array lalu memanggil `txt_zip` untuk diubah menjadi txt dan zip.

Main :
- `int id[]` : array untuk menyimpan id masing-masing manhwa.
- `for (int i = 0; i < 4; i++) manhwa_info(id[i], &heroines[i]);` : melakukan loop untuk mengambil informasi dari manhwa.


### **c. Making the Waifu Gallery**

Mengunduh gambar heroine dari internet dengan jumlah unduhan sesuai dengan bulan rilis manhwa. Gambar disimpan ke dalam subfolder Heroines/(nama heroine).

**Code :**
```
struct Heroine {
    const char *name;
    int count;
    int month;
    const char *img_url;
    char *manhwa_filename;
}

void make_directory(const char *dir_path) {
    pid_t pid = fork();
    if (pid == 0) {
        execlp("mkdir", "mkdir", "-p", dir_path, (char *)NULL);
        perror("execlp failed");
        exit(EXIT_FAILURE);
    } else if (pid < 0) {
        perror("fork failed");
        exit(EXIT_FAILURE);
    } else {
        wait(NULL);  
    }
}

void *download_images(void *arg) {
    struct Heroine *h = (struct Heroine *)arg;

    char dir_path[512];
    snprintf(dir_path, sizeof(dir_path), "Heroines/%s", h->name);
    make_directory("Heroines");
    make_directory(dir_path);

    for (int i = 0; i < h->count; i++) {
        char filename[100];
        snprintf(filename, sizeof(filename), "%s/%s_%d.jpg", dir_path, h->name, i + 1);

        char cmd[1024];
        snprintf(cmd, sizeof(cmd), "wget -q -O \"%s\" \"%s\"", filename, h->img_url);
        system(cmd); 
    }

    archive_images(h->manhwa_filename, h->name);

    pthread_exit(NULL);
}

int main() {
    int id[] = {168827, 147205, 169731, 175521}; 

    struct Heroine heroines[] = {
        {"Lia", 3, "https://i.pinimg.com/736x/11/d9/ad/11d9ad85a47892f5fd979a0209162377.jpg"},
        {"Artezia", 6, "https://i.pinimg.com/736x/cf/ab/a7/cfab73765912c97bc16865df4a9b3455.jpg"},
        {"Adelia", 4, "https://i.pinimg.com/736x/fd/26/a7/fd26a75cbc439e66ed6b55bbd5c904f2.jpg"},
        {"Ophelia", 10, "https://i.pinimg.com/736x/ba/a7/c7/baa7c7b496cc61848d17b471a0000cbd.jpg"}
    };
    
    for (int i = 0; i < 4; i++) {
        manhwa_info(id[i], &heroines[i]);
    }

    pthread_t threads[4];

    for (int i = 0; i < 4; i++) {
        pthread_create(&threads[i], NULL, download_images, &heroines[i]);
        pthread_join(threads[i], NULL);
    }

    cleanup_images();
};
```

**Output :**

Terminal :

![Screenshot from 2025-04-30 15-36-12](https://github.com/user-attachments/assets/f87ff0d4-3eb0-4f91-96aa-41d14aeef51c)

Folder :

![Screenshot from 2025-04-30 15-09-05](https://github.com/user-attachments/assets/464d5220-2397-4ed3-921a-4091fdc1b56a)

Isi folder Heroines :

![Screenshot from 2025-04-30 15-09-15](https://github.com/user-attachments/assets/c1830f4f-b5cb-4cde-9864-648190a8293d)

Isi subfolder Adelia :

![Screenshot from 2025-04-30 15-09-24](https://github.com/user-attachments/assets/fdf61177-cbb9-46c4-9f48-5953ab5c2862)

Isi subfolder Artezia :

![Screenshot from 2025-04-30 15-21-38](https://github.com/user-attachments/assets/f515cf01-dd0a-427e-a219-99199b90cfce)

Isi subfolder Lia :

![Screenshot from 2025-04-30 15-14-50](https://github.com/user-attachments/assets/98e19f71-e330-4f74-bee6-18e49e3b0ae3)

Isi subfolder Ophelia :

![Screenshot from 2025-04-30 15-15-05](https://github.com/user-attachments/assets/e567cb7b-b391-4cbf-80b2-b3c04ec9a4ee)

**Penjelasan :**

Function :

`struct Heroine` :
- Menyimpan data heroine: nama, jumlah gambar, bulan rilis, URL gambar, dan nama file manhwa.

`void make_directory(const char *dir_path)` :
- Membuat folder dengan fork + execlp.

`void *download_images(void *arg)` :
- Fungsi thread untuk mendownload gambar heroine melalui link yang sudah diset di `struct Heroine` yang sudah didefinisikan di `main`
- Membuat folder `Heroines/(nama heroine)`
- Menyimpan gambar dengan `wget`.
- Memanggil fungsi zip gambar heroine (untuk soal d).

Main :
- `struct Heroine heroines[]` : Inisialisasi struct Heroine di main lalu mengisinya dengan data.
- `{"Lia", 3, "https://i.pinimg.com/736x/11/d9/ad/11d9ad85a47892f5fd979a0209162377.jpg"}`. `Lia` : nama heroine, `3` : bulan rilis, `"https://i.pinimg.com/736x/11/d9/ad/11d9ad85a47892f5fd979a0209162377.jpg"` : link gambar.
- `pthread_t threads[4]` : membuat array untuk 4 thread.
- ```
  for (int i = 0; i < 4; i++) {
        pthread_create(&threads[i], NULL, download_images, &heroines[i]);
        pthread_join(threads[i], NULL);
    }
  ```
  Melakukan looping membuat thread baru per heroine untuk memanggil fungsi `download_images`. Satu thread dibuat hingga selesai baru lanjut ke thread selanjutnya.


### **d. Zip. Save. Goodbye**

Mengarsipkan semua gambar heroine, lalu menyimpan ke dalam folder Archive/Images. Sedangkan gambar heroine di folder Heroine/(nama heroine) dihapus sesuai urutan abjad.

**Code :**
```
struct Heroine {
    const char *name;
    int count;
    int month;
    const char *img_url;
    char *manhwa_filename;
}

void make_directory(const char *dir_path) {
    pid_t pid = fork();
    if (pid == 0) {
        execlp("mkdir", "mkdir", "-p", dir_path, (char *)NULL);
        perror("execlp failed");
        exit(EXIT_FAILURE);
    } else if (pid < 0) {
        perror("fork failed");
        exit(EXIT_FAILURE);
    } else {
        wait(NULL);  
    }
}

char *initial_filename(const char *filename) {
    char *initial = malloc(16);
    int j = 0;
    if (isalpha(filename[0])) {
        initial[j++] = toupper(filename[0]);
    }
    for (int i = 1; filename[i]; i++) {
        if (filename[i - 1] == '_' && isalpha(filename[i])) {
            initial[j++] = toupper(filename[i]);
        }
    }
    initial[j] = '\0';
    return initial;
}

void archive_images(const char *manhwa_filename, const char *heroine_name);

void *download_images(void *arg) {
...
    archive_images(h->manhwa_filename, h->name);
...
}

void archive_images(const char *manhwa_filename, const char *heroine_name) {
    make_directory("Archive/Images");
    char *zip_name = initial_filename(manhwa_filename);

    char zip_path_images[512];
    snprintf(zip_path_images, sizeof(zip_path_images), "Archive/Images/%s_%s.zip", zip_name, heroine_name);

    char source_folder[512];
    snprintf(source_folder, sizeof(source_folder), "Heroines/%s", heroine_name);

    char *args[] = {"zip", "-r", "-j", zip_path_images, source_folder, NULL};

    pid_t pid = fork();
    if (pid == 0) {
        execvp("zip", args);
        perror("exec failed");
        exit(EXIT_FAILURE);
    } else if (pid > 0) {
        wait(NULL);
        printf("%s_%s.zip is completed!\n", zip_name, heroine_name);
    } else {
        perror("fork failed");
    }
}

void cleanup_images() {
    DIR *dir;
    struct dirent *entry;
    char *folders[100];
    int count = 0;

    dir = opendir("Heroines");
    if (!dir) {
        perror("opendir failed");
        return;
    }

    while ((entry = readdir(dir)) != NULL) {
        if (entry->d_type == DT_DIR) {
            if (strcmp(entry->d_name, ".") != 0 && strcmp(entry->d_name, "..") != 0) {
                folders[count++] = strdup(entry->d_name);
            }
        }
    }
    closedir(dir);

    for (int i=0; i<count-1; i++) {
        for (int j=i+1; j<count; j++) {
            if (strcmp(folders[i], folders[j]) > 0) {
                char *tmp = folders[i];
                folders[i] = folders[j];
                folders[j] = tmp;
            }
        }
    }

    for (int i = 0; i < count; i++) {
        pid_t pid = fork();
        if (pid == 0) {
            char folder_path[512];
            snprintf(folder_path, sizeof(folder_path), "Heroines/%s", folders[i]);
            execlp("find", "find", folder_path, "-type", "f", "-name", "*.jpg", "-delete", (char *)NULL);
            perror("execlp find failed");
            exit(EXIT_FAILURE);
        } else if (pid > 0) {
            wait(NULL);
            printf("%s images in Heroines/%s deleted successfully!\n", folders[i], folders[i]);
        } else {
            perror("fork failed");
        }
        free(folders[i]);
    }
}

int main() {
    int id[] = {168827, 147205, 169731, 175521}; 

    struct Heroine heroines[] = {
        {"Lia", 3, "https://i.pinimg.com/736x/11/d9/ad/11d9ad85a47892f5fd979a0209162377.jpg"},
        {"Artezia", 6, "https://i.pinimg.com/736x/cf/ab/a7/cfab73765912c97bc16865df4a9b3455.jpg"},
        {"Adelia", 4, "https://i.pinimg.com/736x/fd/26/a7/fd26a75cbc439e66ed6b55bbd5c904f2.jpg"},
        {"Ophelia", 10, "https://i.pinimg.com/736x/ba/a7/c7/baa7c7b496cc61848d17b471a0000cbd.jpg"}
    };
    
    for (int i = 0; i < 4; i++) {
        manhwa_info(id[i], &heroines[i]);
    }

    pthread_t threads[4];

    for (int i = 0; i < 4; i++) {
        pthread_create(&threads[i], NULL, download_images, &heroines[i]);
        pthread_join(threads[i], NULL);
    }

    cleanup_images();
};
```

**Output :**

Terminal :

![Screenshot from 2025-04-30 15-37-13](https://github.com/user-attachments/assets/0a201f07-b860-49ae-a37e-8570f16ee165)

Isi folder Heroines :


Isi folder Archive/Images :

![Screenshot from 2025-04-30 15-39-59](https://github.com/user-attachments/assets/6df45c07-4fa2-4263-a085-b8245e173c18)


**Penjelasan :**

Function :

`struct Heroine` :
- Menyimpan data heroine: nama, jumlah gambar, bulan rilis, URL gambar, dan nama file manhwa.

`void make_directory(const char *dir_path)` :
- Membuat folder dengan fork + execlp.

`void *download_images(void *arg)` :
- Memanggil fungsi zip gambar heroine.

`void archive_images(const char *manhwa_filename, const char *heroine_name)`:
- Membuat directory `Archive/Images`.
- Mengambil nama zip dengan memanggil `initial_filename(manhwa_filename)` lalu menyimpannya ke dalam `zip_name`
- Menyusun path untuk menyimpan file zip dan source folder.
- Melakukan zip dengan fork dan exec.

`void cleanup_images()` :
- Membuka folder `Heroines`, membaca isinya lalu disimpan ke dalam array `folders` , lalu menutup foldernya.
- Mengurutkan nama file yang akan dihapus dengan bubble sort.
- Melakukan loop menghapus file dengan fork() dan exec().

Main :
- `cleanup_images();` : memanggil fungsi `cleanup_images` untuk menghapus gambar heroine urut sesuai abjad.

Kendala pengerjaan :
- Soal c sempat bermasalah pada download (yang terdownload terkadang random).
- Soal d sempat bermasalah pada urutan hapus.
