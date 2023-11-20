# Education NextJS - Route Handler & Authentication

## Table of Content

- [Disclaimer](#disclaimer)
- [Scope Pembelajaran](#scope-pembelajaran)
- [Demo](#demo)
  - [Step 0a - Membuat Collection pada Atlas](#step-0a---membuat-collection-pada-atlas)
  - [Step 0b - Membuat Konfigurasi Driver MongoDB](#step-0b---membuat-konfigurasi-driver-mongodb)
  - [Step 1 - Membuat Kerangka Route Handler `/api/users`](#step-1---membuat-kerangka-route-handler-apiusers)
  - [Step 2 - Membuat Kerangka Route Handler `/api/users/:id`](#step-2---membuat-kerangka-route-handler-apiusersid)
  - [Step 3 - Mengimplementasikan `GET /api/users`](#step-3---mengimplementasikan-get-apiusers)
- [References](#references)

## Disclaimer

- Pada pembelajaran ini kita akan menggunakan `MongoDB` sebagai database untuk melakukan authentication. Untuk itu, kita akan menggunakan `MongoDB Atlas` sebagai layanan database yang akan kita gunakan.

  Pastikan sudah memiliki akun terlebih dahulu di [MongoDB Atlas](https://www.mongodb.com/cloud/atlas).

- Pembelajaran ini menggunakan kode dari pembelajaran sebelumnya yang sudah dimodifikasi sedikit yah. Jadi jangan kaget bila starter code-nya berbeda dengan end code pada pembelajaran sebelumnya

- Pembelajaran ini menggunakan package `jsonwebtoken` sebagai pembuat JWT yang akan disimpan pada cookies, namun, apabila menggunakan `Vercel`, sebenarnya lebih disarankan untuk menggunakan package `jose` yah !

## Scope Pembelajaran

- Route Handler
- Middleware
- Authentication in NextJS

## Demo

Sampai pada titik ini kita sudah menggunakan NextJS pada sisi "FrontEnd" untuk melakukan fetching data dan memutasikan data.

Pada pembelajaran kali ini kita akan menggunakan NextJS pada sisi "BackEnd" untuk membuat API sederhana sampai dengan membuat `Authentication` dengan NextJS yah.

### Step 0a - Membuat Collection pada Atlas

Pada langkah ini kita akan membuat Collection pada MongoDB Atlas dan melakukan seeding data pada collection tersebut secara manual (manual entry via Atlas)

1. Membuka browser dan menuju ke [MongoDB Atlas](https://www.mongodb.com/cloud/atlas)
1. Login ke akun MongoDB Atlas dan membuka Project yang akan digunakan
1. Pilih database yang digunakan kemudian pilih `Browse Collections`
1. Pilih `+ Create Database` (bila sudah memiliki database) atau `Add My Own Data` (bila belum memiliki database)
1. Ketik database name `pengembangan`
1. Ketik collection name `Users`
1. Pada kolom `Additional Preferences`, akan kita kosongkan
1. Tekan tombol `create`

   ![01.png](./assets/01.png)

1. Pada halaman collection `Users` yang sudah dibuat, pilih `INSERT DOCUMENT`
1. Pada modal `Insert Document ini`, pilih `VIEW` yang berbentuk `{}`, kemudian masukkan data di bawah ini pada kolom tersebut

   ```json
   [
     {
       "_id": { "$oid": "655b16efb33bbc4838928d5a" },
       "username": "developer",
       "email": "developer@mail.com",
       "password": "$2a$10$3oBIQBujHQ61vLHn.Nxk..h9SS29vYihvE9CqrPi6yTxFYm1uixgm"
     },
     {
       "_id": { "$oid": "655b17c3aef0d8a5d9d180d9" },
       "username": "admin",
       "email": "admin@mail.com",
       "password": "$2a$10$6OhSE/SgQQPWLe3VFtnNZuZcpu4x445yJm5cgdVZxfb6AYpCKtdkC",
       "superadmin": true
     },
     {
       "_id": { "$oid": "655b17fc36dce13753b7efda" },
       "username": "other",
       "email": "other@mail.com",
       "password": "$2a$10$6SA9Z5J8aD0QyuQSfFBC4uMb2JIfHnpJHKN75V0q0zIELlxWQzK9W",
       "original_name": "Just Another"
     }
   ]
   ```

1. Tekan tombol `Insert`

   ![02.png](./assets/02.png)

Sampai pada tahap ini, kita sudah berhasil untuk memasukkan data yang dimiliki ke dalam Atlas yah.

### Step 0b - Membuat Konfigurasi Driver MongoDB

Pada langkah ini kita akan membuat konfigurasi awal untuk menggunakan driver mongodb agar dapat terkoneksi dengan Atlas via package `mongodb`.

Adapun langkah-langkahnya adalah sebagai berikut:

1. Membuka [halaman utama Atlas](https://cloud.mongodb.com/)
1. Menekan tombol `Connect` kemudian memilih `Drivers`
1. Pada langkah `3. Add your connection string into your application code`, akan diberikan sebuah string yang diawali dengan `mongo+srv`, tekan tombol copy
1. Kembali pada halaman project pada VSCode, membuat sebuah file baru dengan nama `.env` pada root folder (`sources/a-start/client/.env`)
1. Membuat sebuah key baru dengan nama `MONGODB_CONNECTION_STRING="<isikan_dengan_string_yang_dicopy_tadi>"` (**perhatikan bahwa ada double quote pada string tersebut**)
1. Membuat sebuah key baru dengan nama `MONGODB_DB_NAME=pengembangan`
1. Menginstall package `mongodb` dengan perintah `npm install mongodb`
1. Menginstall package `bcrypt` dengan perintah `npm install bcryptjs`
1. Menginstall type definition `bcrypt` dengan perintah `npm install -D @types/bcryptjs`
1. Membuat folder baru pada `src` dengan nama `db` (`/src/db`)
1. Membuat folder baru pada `src/db` dengan nama `config` dan `models` (`/src/db/config` dan `/src/db/models`)
1. Membuat file baru dengan nama `index.ts` pada folder `config` (`src/config/index.ts`) dan menuliskan kode sebagai berikut:

   ```ts
   import { MongoClient } from "mongodb";

   const connectionString = process.env.MONGODB_CONNECTION_STRING;

   // Memastikan bahwa connectionString sudah ada value-nya
   if (!connectionString) {
     throw new Error("MONGODB_CONNECTION_STRING is not defined");
   }

   // Tipe data dari client adalah MongoClient
   let client: MongoClient;

   // Fungsi ini akan mengembalikan client yang sudah terkoneksi dengan MongoDB
   // Hanya boleh ada 1 instance client (Singleton)
   export const getMongoClientInstance = async () => {
     if (!client) {
       client = await MongoClient.connect(connectionString);
       await client.connect();
     }

     return client;
   };
   ```

1. Membuat sebuah file baru dengan nama `user.ts` pada folder `models` (`src/models/user.ts`) dan menuliskan kode sebagai berikut:

   ```ts
   import { Db, ObjectId } from "mongodb";
   import { getMongoClientInstance } from "../config";
   import { hashText } from "../utils/hash";

   // Mendefinisikan type dari UserModel
   type UserModel = {
     _id: ObjectId;
     username: string;
     email: string;
     password: string;
     // Perhatikan di sini menggunakan ? (optional)
     // Karena tidak semua data yang ada di dalam collection memiliki field ini
     superadmin?: boolean;
     original_name?: string;
   };

   // constant value
   const DATABASE_NAME = process.env.MONGODB_DATABASE_NAME || "test";
   const COLLECTION_USER = "Users";

   // Model CRUD
   export const getDb = async () => {
     const client = await getMongoClientInstance();
     const db: Db = client.db(DATABASE_NAME);

     return db;
   };

   export const getUsers = async () => {
     const db = await getDb();

     // Di sini kita akan mendefinisikan type dari users
     // Karena kembalian dari toArray() adalah array `WithId<Document>[]`
     // kita akan type casting menjadi UserModel[] dengan menggunakan "as"
     const users = (await db
       .collection(COLLECTION_USER)
       .find({})
       // Exclude kolom password
       // (For the sake of security...)
       .project({ password: 0 })
       .toArray()) as UserModel[];

     return users;
   };

   export const createUser = async (user: UserModel) => {
     // Kita akan memodifikasi user yang baru
     // karena butuh untuk meng-hash password
     // (For the sake of security...)
     const modifiedUser: UserModel = {
       ...user,
       password: hashText(user.password),
     };

     const db = await getDb();
     const result = await db
       .collection(COLLECTION_USER)
       .insertOne(modifiedUser);

     return result;
   };

   export const findUserByEmail = async (email: string) => {
     const db = await getDb();

     const user = (await db
       .collection(COLLECTION_USER)
       .findOne({ email: email })) as UserModel;

     return user;
   };
   ```

Sampai pada tahap ini artinya kita sudah siap untuk membuat sisi "BackEnd" dari NextJS yang cukup sederhana yah !

### Step 1 - Membuat Kerangka Route Handler `/api/users`

Pada langkah ini kita akan mencoba untuk membuat kerangka untuk route `/api/users`, yaitu:

- `GET /api/users` untuk mendapatkan semua data user
- `POST /api/users` untuk membuat user baru

Untuk bisa membuat `endpoint` seperti ini, kita akan menggunakan `Route Handlers` yang sudah disediakan oleh NextJS.

`Route Handlers` didefinisikan di dalam NextJS dalam file bernama `route.js` atau `route.ts` yang berada di dalam folder `app`, dan di dalam file tersebut kita bisa mendefinisikan `endpoint` yang akan kita buat via fungsi dengan nama HTTP method yang diinginkan dalam `CAPSLOCK` a.k.a `GET`, `POST`, `PUT`, `DELETE`, dan lain-lain.

```ts
// GET /api/nama_resource/route.ts
export const GET = () => {
  // Do some magic here
};

// POST /api/nama_resource/route.ts
export const POST = () => {
  // Do some magic here
};
```

> Bagi yang menggunakan `pages` untuk membuat `endpoint`, maka pembelajaran ini bukan yang paling tepat untuk Anda yah...

Adapun langkah-langkah pembuatannya adalah sebagai berikut:

1. Membuat sebuah folder baru pada folder `app` dengan nama `api` (`src/app/api`)
1. Membuat sebuah folder baru di dalam `api` dengan nama `users` (`src/app/api/users`)
1. Membuat sebuah file baru dengan nama `route.ts` (`src/app/api/users/route.ts`) dan memasukkan kode sebagai berikut:

   ```ts
   // Di sini kita akan mengimport NextResponse
   // Untuk penjelasannya ada di bawah yah !
   import { NextResponse } from "next/server";

   // Type definitions untuk Response yang akan dikembalikan
   type MyResponse<T> = {
     statusCode: number;
     message?: string;
     data?: T;
     error?: string;
   };

   // GET /api/users
   export const GET = async () => {
     // Di sini yang akan dikembalikan adalah Response dari Web API
     // (Standard Web API: Request untuk mendapatkan data dan Request untuk mengirimkan data)
     // https://developer.mozilla.org/en-US/docs/Web/API/Request
     // https://developer.mozilla.org/en-US/docs/Web/API/Response
     return Response.json(
       // Data yang akan dikirimkan ke client
       {
         statusCode: 200,
         message: "Pong from GET /api/users !",
       },
       // Object informasi tambahan (status code, headers, dll)
       {
         // Default status adalah 200
         status: 200,
       }
     );
   };

   // POST /api/users
   export const POST = async () => {
     // Di sini kita akan menggunakan NextResponse yang merupakan extend dari Response
     // Keuntungan dengan menggunakan NextResponse adalah kita bisa menuliskan kembalian dari Response dengan lebih presisi dengan Generic Type dan memiliki beberapa method yang tidak ada di Response.
     // https://nextjs.org/docs/pages/api-reference/functions/next-server#nextresponse
     // Misalnya di sini kita menuliskan bahwa Response yang akan dikembalikan adalah MyResponse yang mana memiliki Generic Type never (tidak ada data yang dikembalikan) untuk key "data"

     /*
        {
          statusCode: number; <--- harus selalu ada statusCode
          message?: string; <--- bisa ada "message" bisa tidak
          data?: never; <--- menjadi tidak ada "data" yang dikembalikan
          error?: string; <--- bisa ada "error" bisa tidak
        }
      */
     return NextResponse.json<MyResponse<never>>(
       // Data yang akan dikirimkan ke client
       {
         statusCode: 201,
         message: "Pong from POST /api/users !",
       },
       // Object informasi tambahan (status code, headers, dll)
       {
         // Karena di sini menggunakan non-default status (bukan 200)
         // maka di sini kita menuliskan status: 201
         status: 201,
       }
     );
   };
   ```

1. Menjalankan aplikasi dengan perintah `npm run dev`
1. Membuka HTTP REST Client yang bisa digunakan seperti [Postman](https://www.postman.com/) atau [Insomnia](https://insomnia.rest/) atau [VSCode REST Client](https://marketplace.visualstudio.com/items?itemName=humao.rest-client) dan cobalah untuk menembak pada endpoint berikut:

   - `GET http://localhost:3000/api/users`
   - `POST http://localhost:3000/api/users`

   Dan lihatlah hasilnya, apakah sesuai dengan json yang dibuat?

### Step 2 - Membuat Kerangka Route Handler `/api/users/:id`

Pada langkah ini kita akan mencoba untuk membuat kerangka untuk route `/api/users/:id`, yaitu:

- `GET /api/users/:id` untuk mendapatkan data user berdasarkan id

Adapun langkah-langkah pembuatannya adalah sebagai berikut:

1. Membuat sebuah folder baru pada folder `users` dengan nama `[id]` (`src/app/api/users/[id]`)
1. Membuat sebuah file baru dengan nama `route.ts` (`src/app/api/users/[id]/route.ts`) dan menuliskan kode sebagai berikut:

   ```ts
   // di sini kita akan menggunakan NextRequest dan NextResponse yang merupakan extend dari Request dan Response
   import { NextRequest, NextResponse } from "next/server";

   // Type definitions untuk Response yang akan dikembalikan
   type MyResponse<T> = {
     statusCode: number;
     message?: string;
     data?: T;
     error?: string;
   };

   // Karena di sini kita akan menerima parameter "id" dari URL
   // Maka di sini kita akan menggunakan parameter kedua dari route handler
   // yaitu berupa suatu Object params
   export const GET = (
     // Di sini kita menggunakan _request karena kita tidak akan menggunakan argument ini sekarang
     _request: NextRequest,
     // Perhatikan di sini params dalam bentuk sebuah Object
     { params }: { params: { id: string } }
   ) => {
     const id = params.id;

     return NextResponse.json<MyResponse<unknown>>({
       statusCode: 200,
       message: `Pong from GET /api/users/${id} !`,
     });
   };
   ```

1. Membuka HTTP Rest Client dan menembak ke endpoint `GET http://localhost:3000/api/users/123` dan lihatlah hasilnya

   Apakah sesuai dengan json yang dibuat?

### Step 3 - Mengimplementasikan `GET /api/users`

## References

- https://nextjs.org/docs/app/building-your-application/routing/route-handlers
- https://nextjs.org/docs/pages/api-reference/functions/next-server#nextrequest
- https://nextjs.org/docs/pages/api-reference/functions/next-server#nextresponse
