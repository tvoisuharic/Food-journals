# Food Journal App

A React Native application for tracking food entries with image and category support.

## Setup Instructions

1. Install dependencies:

```bash
npm install
```

2. Start the development server:

```bash
npm start
```

# Debugging and Fixes

During the development of the Food Journal application, several significant issues arose that required careful analysis and correction. The main difficulties were related to initializing the database, executing SQL queries, and working with user interface components such as ScrollView and SwipeListView.

## Issues and Solutions

### 1. Database Initialization Error

**Error:**
Database initialization error: `[TypeError: Cannot read property 'execAsync' of undefined]`

**Root Cause:**
The `execAsync` method was being called on an undefined object (`tx`). In newer versions of expo-sqlite, the `execAsync` method can only be called directly on the database instance (`db`) and not within a transaction context.

**Solution:**
Replaced the incorrect code with the correct implementation:

**Incorrect Code:**
```javascript
await db.withTransactionAsync(async (tx) => {
  await tx.execAsync(
    `CREATE TABLE IF NOT EXISTS users (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      email TEXT UNIQUE,
      password TEXT
    );`
  );
});
```

**Сorrect Code:**
```javascript
await db.withTransactionAsync(async () => {
  await db.execAsync(
    `CREATE TABLE IF NOT EXISTS users (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      email TEXT UNIQUE,
      password TEXT
    );`
  );
});
```
## **2. Query Execution Error in `authScreen.js`**

### **Error**
```plaintext
Database error: [TypeError: Cannot read property 'rows' of undefined]
```

### **Root Cause**
The `checkResult` object was undefined because the `executeSql` method did not return a valid result. Additionally, the `execAsync` method in newer versions of `expo-sqlite` does not support parameterized queries. Instead, the `getAllAsync` method should be used for queries that return rows.

### **Solution**
Refactored the SQL execution logic to use `getAllAsync` for SELECT queries.

#### **Old Query Execution Method**
```js
const executeSql = async (query, params = []) => {
  try {
    if (!isInitialized) {
      await initDatabase();
    }
    return await db.withTransactionAsync(async () => {
      return await db.execAsync(query, params);
    });
  } catch (error) {
    console.error('SQL execution error:', error);
    throw error;
  }
};
```

#### **Corrected Query Execution Method**
```js
const executeQuerySql = async (query, params = []) => {
  try {
    if (!isInitialized) {
      await initDatabase();
    }
    return await db.withTransactionAsync(async () => {
      return await db.getAllAsync(query, params);
    });
  } catch (error) {
    console.error('SQL execution error:', error);
    throw error;
  }
};
```

#### **Updated Code in `authScreen.js`**
```js
const checkResult = await executeQuerySql(
  'SELECT id FROM users WHERE email = ?',
  [email]
);

console.log('Query result:', checkResult);

if (checkResult.length > 0) {
  Alert.alert('Registration Failed', 'Email already exists');
  return;
}
```

---

## **3. DML Operations (INSERT/UPDATE/DELETE)**

### **Error**
The `runAsync` method was required for executing Data Manipulation Language (DML) operations like INSERT, but the existing `executeSql` method did not support it.

### **Solution**
Created a new method `executeCURDSql` specifically for DML operations.

#### **New Method for DML Operations**
```js
const executeCURDSql = async (query, params = []) => {
  try {
    if (!isInitialized) {
      await initDatabase();
    }
    return await db.withTransactionAsync(async () => {
      return await db.runAsync(query, params);
    });
  } catch (error) {
    console.error('SQL DML execution error:', error);
    throw error;
  }
};
```

#### **Usage in `authScreen.js`**
```js
const insertResult = await executeCURDSql(
  'INSERT INTO users (email, password) VALUES (?, ?)',
  [email, password]
);

console.log('Insert result lastId:', insertResult.lastInsertRowId, insertResult.changes);

navigation.navigate('Home', { userId: insertResult.lastInsertRowId });
```

---

## **4. Authentication Check Error**

### **Error**
```plaintext
TypeError: Cannot read property 'item' of undefined
```

### **Root Cause**
In newer versions of `expo-sqlite`, query results are returned as arrays instead of objects with a `rows` property. The `item` method is no longer available.

### **Solution**
Updated the authentication check logic to handle array-based results.

#### **Incorrect Code**
```js
if (result.length > 0) {
  navigation.navigate('Home', { userId: result.rows.item(0).id });
} else {
  Alert.alert('Authentication Failed', 'Invalid email or password');
}
```

#### **Corrected Code**
```js
if (result.length > 0) {
  navigation.navigate('Home', { userId: result[0]['id'] });
} else {
  Alert.alert('Authentication Failed', 'Invalid email or password');
}
```

---

## **5. HomeScreen Issues**

### **Issue 1: Persistent Loading State**
The `HomeScreen` remained stuck in a loading state (`isLoading`). This was likely due to improper initialization of components like camera permissions.

#### **Solution**
Verified and updated the camera permission logic to match the latest `expo-camera` API.

#### **Updated Code**
```js
console.log('Initializing...');
if (!cameraPermission || cameraPermission.status !== 'granted') {
  console.log('Requesting camera permission...');
  try {
    await requestCameraPermission();
  } catch (error) {
    console.error('Error requesting camera permission:', error);
    Alert.alert('Error', 'Failed to request camera permission');
  }
} else {
  console.log('Camera permission status:', cameraPermission.status);
}
setHasCameraPermission(cameraPermission?.status === 'granted');
```

---

### **Issue 2: Journals List Not Displaying**
The journals list was not visible and could not be scrolled due to conflicts between `ScrollView` and `SwipeListView`.

#### **Error**
```plaintext
VirtualizedLists should never be nested inside plain ScrollViews with the same orientation because it can break windowing and other functionality - use another VirtualizedList-backed container instead.
```

#### **Solution**
Split the `HomeScreen` into two separate screens:
- **JournalsListScreen**: Displays the list of journals.
- **JournalManagerScreen**: Handles adding and editing journals.

#### **Refactored Code**
```jsx
<SwipeListView
              data={filteredJournals}
              keyExtractor={(item) => item.id.toString()}
              renderItem={({ item }) => (
                <View style={styles.journalItem}>
                  <Image source={{ uri: item.image }} style={styles.journalImage} />
                  <View style={styles.journalDetails}>
                    <Text style={styles.journalDescription}>
                      {item.description}
                    </Text>
                    <View style={styles.journalMeta}>
                      <Text style={styles.journalCategory}>
                        {item.category}
                      </Text>
                      <Text style={styles.journalDate}>
                        {new Date(item.date).toLocaleDateString()}
                      </Text>
                    </View>
                  </View>
                </View>
              )}
              renderHiddenItem={({ item }) => (
                <View style={styles.hiddenButtons}>
                  <TouchableOpacity
                    style={[styles.hiddenButton, styles.editButton]}
                    onPress={() => {
                      editJournal(item.id, item.description, item.image, item.category);
                    }}
                  >
                    <Text style={styles.hiddenButtonText}>Edit</Text>
                  </TouchableOpacity>
                  <TouchableOpacity
                    style={[styles.hiddenButton, styles.deleteButton]}
                    onPress={() => deleteJournal(item.id)}
                  >
                    <Text style={styles.hiddenButtonText}>Delete</Text>
                  </TouchableOpacity>
                </View>
              )}
              rightOpenValue={-150}
              disableRightSwipe
              showsVerticalScrollIndicator={false}
            />
  rightOpenValue={-150}
  disableRightSwipe
  showsVerticalScrollIndicator={false}
/>
```


