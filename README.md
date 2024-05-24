# Clean Architecture in Vue.js con Composition API

## Struttura del Progetto

```plaintext
src/
│
├── actions/
│   └── userActions.js
│
├── usecases/
│   ├── fetchUserData.js
│   ├── updateUserData.js
│   └── deleteUserData.js
│
├── entities/
│   └── User.js
│
├── dataaccess/
│   ├── api.js
│   └── userRepository.js
│
├── components/
│   ├── UserComponent.vue
│   ├── UserStatusComponent.vue
│   ├── UserSaveComponent.vue
│   └── UserDeleteComponent.vue
│
└── App.vue
```

## 1. Actions

Le Actions sono responsabili di preparare i dati e passare gli input agli use cases. Ecco alcune action più complesse.

```javascript
// src/actions/userActions.js
import { fetchUserData } from '../usecases/fetchUserData';
import { updateUserData } from '../usecases/updateUserData';
import { deleteUserData } from '../usecases/deleteUserData';

export async function loadAndCheckUserActive(userId) {
  const userData = await fetchUserData(userId);
  return userData.isActive;
}

export async function saveUserAndConfirm(userDto) {
  try {
    await updateUserData(userDto);
    return true;
  } catch (error) {
    console.error('Error saving user data:', error);
    return false;
  }
}

export async function deleteUserAndConfirm(userId) {
  try {
    await deleteUserData(userId);
    return true;
  } catch (error) {
    console.error('Error deleting user data:', error);
    return false;
  }
}

export async function loadUserDataAndRenderMessage(userId) {
  const userData = await fetchUserData(userId);
  
  if (userData.isActive) {
    return 'User is active';
  } else {
    return 'User is inactive';
  }
}
```

## 2. Use Cases

Gli Use Cases contengono la logica di business e chiamano il livello di Data Access per ottenere, aggiornare o cancellare i dati. Utilizzano le entità per controllare o modificare i dati e poi mappano l'entità in DTO.

```javascript
// src/usecases/fetchUserData.js
import { getUserById } from '../dataaccess/userRepository';
import { User } from '../entities/User';

export async function fetchUserData(userId) {
  const userDto = await getUserById(userId);
  const userEntity = new User(userDto);

  // Esempio di validazione o modifica dei dati
  userEntity.validate();
  userEntity.updateName(userEntity.name.toUpperCase());

  // Mappare l'entità in DTO prima di ritornare
  return userEntity.toDto();
}
```

```javascript
// src/usecases/updateUserData.js
import { saveUser } from '../dataaccess/userRepository';
import { User } from '../entities/User';

export async function updateUserData(userDto) {
  const userEntity = new User(userDto);

  // Esempio di validazione o modifica dei dati
  userEntity.validate();

  // Salvare i dati aggiornati
  const updatedUserDto = await saveUser(userEntity.toDto());
  return updatedUserDto;
}
```

```javascript
// src/usecases/deleteUserData.js
import { deleteUser } from '../dataaccess/userRepository';

export async function deleteUserData(userId) {
  await deleteUser(userId);
}
```

## 3. Entity

Le Entity sono utilizzate per controllare e modificare i dati nello use case e poi mappare i dati in un DTO.

```javascript
// src/entities/User.js
export class User {
  constructor({ id, name, email, isActive }) {
    this.id = id;
    this.name = name;
    this.email = email;
    this.isActive = isActive;
  }

  validate() {
    if (!this.email.includes('@')) {
      throw new Error('Invalid email format');
    }
  }

  updateName(newName) {
    this.name = newName;
  }

  toDto() {
    return {
      id: this.id,
      name: this.name,
      email: this.email,
      isActive: this.isActive,
    };
  }
}
```

## 4. Data Access

Il livello di Data Access è responsabile delle chiamate API e dei mapper che trasformano i dati in DTO.

```javascript
// src/dataaccess/api.js
export async function fetchUserFromApi(userId) {
  const response = await fetch(`/api/users/${userId}`);
  return response.json();
}

export async function saveUserToApi(userDto) {
  const response = await fetch(`/api/users/${userDto.id}`, {
    method: 'PUT',
    headers: {
      'Content-Type': 'application/json',
    },
    body: JSON.stringify(userDto),
  });
  return response.json();
}

export async function deleteUserFromApi(userId) {
  await fetch(`/api/users/${userId}`, {
    method: 'DELETE',
  });
}
```

```javascript
// src/dataaccess/userRepository.js
import { fetchUserFromApi, saveUserToApi, deleteUserFromApi } from './api';

export async function getUserById(userId) {
  const userData = await fetchUserFromApi(userId);
  return mapUserDataToDto(userData);
}

export async function saveUser(userDto) {
  const updatedUserData = await saveUserToApi(userDto);
  return mapUserDataToDto(updatedUserData);
}

export async function deleteUser(userId) {
  await deleteUserFromApi(userId);
}

function mapUserDataToDto(data) {
  return {
    id: data.id,
    name: data.name,
    email: data.email,
    isActive: data.isActive,
  };
}
```

## 5. Componenti

Ecco come utilizzare queste azioni all'interno di componenti Vue.

```vue
<!-- src/components/UserStatusComponent.vue -->
<template>
  <div>
    <div v-if="userActive">
      <p>User is active</p>
    </div>
    <div v-else>
      <p>User is inactive</p>
    </div>
  </div>
</template>

<script>
import { ref, onMounted } from 'vue';
import { loadAndCheckUserActive } from '../actions/userActions';

export default {
  setup() {
    const userActive = ref(false);

    onMounted(async () => {
      userActive.value = await loadAndCheckUserActive(1);
    });

    return { userActive };
  },
};
</script>
```

```vue
<!-- src/components/UserSaveComponent.vue -->
<template>
  <div>
    <button @click="saveUser">Save User</button>
    <p v-if="saveConfirmed">User saved successfully!</p>
    <p v-else-if="saveAttempted && !saveConfirmed">Failed to save user.</p>
  </div>
</template>

<script>
import { ref } from 'vue';
import { saveUserAndConfirm } from '../actions/userActions';

export default {
  setup() {
    const saveConfirmed = ref(false);
    const saveAttempted = ref(false);

    const userDto = { id: 1, name: 'John Doe', email: 'john.doe@example.com', isActive: true };

    const saveUser = async () => {
      saveAttempted.value = true;
      saveConfirmed.value = await saveUserAndConfirm(userDto);
    };

    return { saveUser, saveConfirmed, saveAttempted };
  },
};
</script>
```

```vue
<!-- src/components/UserDeleteComponent.vue -->
<template>
  <div>
    <button @click="deleteUser">Delete User</button>
    <p v-if="deleteConfirmed">User deleted successfully!</p>
    <p v-else-if="deleteAttempted && !deleteConfirmed">Failed to delete user.</p>
  </div>
</template>

<script>
import { ref } from 'vue';
import { deleteUserAndConfirm } from '../actions/userActions';

export default {
  setup() {
    const deleteConfirmed = ref(false);
    const deleteAttempted = ref(false);

    const deleteUser = async () => {
      deleteAttempted.value = true;
      deleteConfirmed.value = await deleteUserAndConfirm(1);
    };

    return { deleteUser, deleteConfirmed, deleteAttempted };
  },
};
</script>
```

Passaggi Spiegati

	1.	Actions: La funzione loadUserData chiama l’use case fetchUserData, che si occupa di ottenere i dati dell’utente.
	2.	Use Cases: La funzione fetchUserData chiama il repository getUserById per ottenere i dati dal database/API, crea un’entità User, esegue le validazioni o modifiche necessarie, e poi mappa l’entità in un DTO.
	3.	Entity: La classe User contiene la logica per la validazione e la modifica dei dati, oltre a un metodo toDto per mappare l’entità in un DTO.
	4.	Data Access: Il repository userRepository utilizza un mapper per convertire i dati ottenuti dall’API in un DTO.
	5.	Component: Il componente Vue utilizza la Composition API per caricare e visualizzare i dati dell’utente al montaggio del componente.
