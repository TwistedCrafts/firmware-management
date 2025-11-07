# firmware-management
management for mobile firmwares
# Firmware Mobile App (TypeScript)

This canvas contains a complete starter React Native (Expo + TypeScript) admin app for firmware management. Drop these files into `mobile/` and run with Expo.

---

## File: package.json

```json
{
  "name": "firmware-admin",
  "version": "0.1.0",
  "private": true,
  "main": "node_modules/expo/AppEntry.js",
  "scripts": {
    "start": "expo start",
    "android": "expo run:android",
    "ios": "expo run:ios",
    "web": "expo start --web"
  },
  "dependencies": {
    "expo": "~48.0.0",
    "expo-document-picker": "~11.0.0",
    "expo-secure-store": "~12.0.0",
    "@react-native-async-storage/async-storage": "~1.17.11",
    "react": "18.2.0",
    "react-native": "0.71.8",
    "@react-navigation/native": "^6.1.6",
    "@react-navigation/native-stack": "^6.9.12",
    "react-native-gesture-handler": "^2.9.0",
    "react-native-screens": "~3.20.0",
    "react-native-paper": "^5.10.0"
  },
  "devDependencies": {
    "typescript": "^5.1.6",
    "@types/react": "^18.2.21",
    "@types/react-native": "^0.71.10"
  }
}
```

---

## File: tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ESNext",
    "module": "ESNext",
    "jsx": "react-jsx",
    "strict": true,
    "moduleResolution": "node",
    "esModuleInterop": true,
    "skipLibCheck": true,
    "resolveJsonModule": true,
    "allowJs": true,
    "noEmit": true
  },
  "exclude": ["node_modules"]
}
```

---

## File: App.tsx

```tsx
import React from 'react';
import { NavigationContainer } from '@react-navigation/native';
import { createNativeStackNavigator } from '@react-navigation/native-stack';
import LoginScreen from './src/screens/LoginScreen';
import Dashboard from './src/screens/Dashboard';
import DevicesScreen from './src/screens/DevicesScreen';
import DeviceDetail from './src/screens/DeviceDetail';
import FirmwaresScreen from './src/screens/FirmwaresScreen';
import UploadFirmware from './src/screens/UploadFirmware';
import CreateCampaign from './src/screens/CreateCampaign';

export type RootStackParamList = {
  Login: undefined;
  Dashboard: undefined;
  Devices: undefined;
  DeviceDetail: { deviceId: string };
  Firmwares: undefined;
  UploadFirmware: undefined;
  CreateCampaign: undefined;
};

const Stack = createNativeStackNavigator<RootStackParamList>();

export default function App() {
  return (
    <NavigationContainer>
      <Stack.Navigator initialRouteName="Login">
        <Stack.Screen name="Login" component={LoginScreen} />
        <Stack.Screen name="Dashboard" component={Dashboard} />
        <Stack.Screen name="Devices" component={DevicesScreen} />
        <Stack.Screen name="DeviceDetail" component={DeviceDetail} />
        <Stack.Screen name="Firmwares" component={FirmwaresScreen} />
        <Stack.Screen name="UploadFirmware" component={UploadFirmware} />
        <Stack.Screen name="CreateCampaign" component={CreateCampaign} />
      </Stack.Navigator>
    </NavigationContainer>
  );
}
```

---

## File: src/auth.ts

```ts
import * as SecureStore from 'expo-secure-store';
const TOKEN_KEY = 'fm_token';

export async function saveToken(token: string) {
  await SecureStore.setItemAsync(TOKEN_KEY, token);
}
export async function getToken(): Promise<string | null> {
  return await SecureStore.getItemAsync(TOKEN_KEY);
}
export async function clearToken() {
  await SecureStore.deleteItemAsync(TOKEN_KEY);
}
```

---

## File: src/api.ts

```ts
import AsyncStorage from '@react-native-async-storage/async-storage';
import { getToken } from './auth';

const API_BASE = process.env.API_BASE || 'http://10.0.2.2:3000';

async function authHeaders() {
  const token = await getToken();
  return { 'Content-Type': 'application/json', ...(token ? { Authorization: `Bearer ${token}` } : {}) };
}

export async function apiGET(path: string) {
  const res = await fetch(`${API_BASE}${path}`, { headers: await authHeaders() });
  if (res.status === 204) return null;
  if (!res.ok) throw new Error(await res.text());
  return res.json();
}

export async function apiPOST(path: string, body: any) {
  const res = await fetch(`${API_BASE}${path}`, { method: 'POST', headers: await authHeaders(), body: JSON.stringify(body) });
  if (!res.ok) throw new Error(await res.text());
  return res.json();
}

export async function presignUpload(filename: string, content_type: string, size: number) {
  return apiPOST('/api/v1/firmwares/presign', { filename, content_type, size });
}
```

---

## File: src/screens/LoginScreen.tsx

```tsx
import React, { useState } from 'react';
import { View, TextInput, Button, Text, StyleSheet } from 'react-native';
import { apiPOST } from '../api';
import { saveToken } from '../auth';

export default function LoginScreen({ navigation }: any) {
  const [username, setUsername] = useState('');
  const [password, setPassword] = useState('');

  async function login() {
    try {
      const res = await apiPOST('/api/v1/login', { username, password });
      await saveToken(res.token);
      navigation.replace('Dashboard');
    } catch (e: any) {
      alert('Login failed: ' + (e.message || e));
    }
  }

  return (
    <View style={styles.container}>
      <Text style={styles.title}>Firmware Admin</Text>
      <TextInput placeholder="Username" value={username} onChangeText={setUsername} style={styles.input} />
      <TextInput placeholder="Password" value={password} onChangeText={setPassword} secureTextEntry style={styles.input} />
      <Button title="Log in" onPress={login} />
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, padding: 16, justifyContent: 'center' },
  title: { fontSize: 24, fontWeight: '700', marginBottom: 12 },
  input: { borderWidth: 1, padding: 8, marginBottom: 12 }
});
```

---

## File: src/screens/Dashboard.tsx

```tsx
import React, { useEffect, useState } from 'react';
import { View, Text, Button, StyleSheet } from 'react-native';
import { apiGET } from '../api';

export default function Dashboard({ navigation }: any) {
  const [stats, setStats] = useState<any>(null);

  useEffect(() => {
    let mounted = true;
    async function load() {
      try {
        const res = await apiGET('/api/v1/dashboard');
        if (mounted) setStats(res);
      } catch (e) {
        console.error(e);
      }
    }
    load();
    return () => { mounted = false; };
  }, []);

  return (
    <View style={styles.container}>
      <Text style={styles.title}>Dashboard</Text>
      <Button title="Devices" onPress={() => navigation.navigate('Devices')} />
      <Button title="Firmwares" onPress={() => navigation.navigate('Firmwares')} />
      <View style={{ marginTop: 16 }}>
        <Text>Active devices: {stats?.active ?? '...'}</Text>
        <Text>Updating now: {stats?.updating ?? '...'}</Text>
        <Text>Failed: {stats?.failed ?? '...'}</Text>
      </View>
    </View>
  );
}

const styles = StyleSheet.create({ container: { flex: 1, padding: 16 }, title: { fontSize: 20, fontWeight: '700', marginBottom: 12 } });
```

---

## File: src/screens/DevicesScreen.tsx

```tsx
import React, { useEffect, useState } from 'react';
import { View, FlatList, Text, TouchableOpacity, StyleSheet } from 'react-native';
import { apiGET } from '../api';

export default function DevicesScreen({ navigation }: any) {
  const [devices, setDevices] = useState<any[]>([]);

  useEffect(() => {
    let mounted = true;
    async function load() {
      try {
        const res = await apiGET('/api/v1/devices');
        if (mounted) setDevices(res);
      } catch (e) { console.error(e); }
    }
    load();
    const t = setInterval(load, 10000);
    return () => { mounted = false; clearInterval(t); };
  }, []);

  return (
    <View style={{ flex: 1 }}>
      <FlatList
        data={devices}
        keyExtractor={(d) => d.device_id}
        renderItem={({ item }) => (
          <TouchableOpacity onPress={() => navigation.navigate('DeviceDetail', { deviceId: item.device_id })}>
            <View style={styles.row}>
              <Text style={{ fontWeight: '600' }}>{item.device_id} — {item.model}</Text>
              <Text>{item.firmware_version} • {item.status || 'unknown'}</Text>
            </View>
          </TouchableOpacity>
        )}
      />
    </View>
  );
}

const styles = StyleSheet.create({ row: { padding: 12, borderBottomWidth: 1, borderColor: '#eee' } });
```

---

## File: src/screens/DeviceDetail.tsx

```tsx
import React, { useEffect, useState } from 'react';
import { View, Text, Button, StyleSheet } from 'react-native';
import { apiGET, apiPOST } from '../api';

export default function DeviceDetail({ route }: any) {
  const { deviceId } = route.params;
  const [device, setDevice] = useState<any>(null);

  useEffect(() => { apiGET(`/api/v1/devices/${deviceId}`).then(setDevice).catch(() => {}); }, []);

  async function triggerUpdate() {
    try {
      await apiPOST(`/api/v1/devices/${deviceId}/trigger`, {});
      alert('Triggered update');
    } catch (e: any) { alert('Failed: ' + (e.message || e)); }
  }

  if (!device) return <Text>Loading...</Text>;
  return (
    <View style={styles.container}>
      <Text style={styles.title}>{device.device_id}</Text>
      <Text>Model: {device.model}</Text>
      <Text>Firmware: {device.firmware_version}</Text>
      <Button title="Trigger Update" onPress={triggerUpdate} />
      <Text style={{ marginTop: 12 }}>Last seen: {device.last_seen}</Text>
    </View>
  );
}

const styles = StyleSheet.create({ container: { padding: 16 }, title: { fontSize: 18, fontWeight: '600' } });
```

---

## File: src/screens/FirmwaresScreen.tsx

```tsx
import React, { useEffect, useState } from 'react';
import { View, FlatList, Text, Button, StyleSheet } from 'react-native';
import { apiGET } from '../api';

export default function FirmwaresScreen({ navigation }: any) {
  const [fw, setFw] = useState<any[]>([]);
  useEffect(() => { apiGET('/api/v1/firmwares').then(setFw).catch(() => {}); }, []);
  return (
    <View style={{ flex: 1, padding: 8 }}>
      <Button title="Upload" onPress={() => navigation.navigate('UploadFirmware')} />
      <FlatList data={fw} keyExtractor={(f) => String(f.id)} renderItem={({ item }) => (
        <View style={styles.row}>
          <Text style={{ fontWeight: '600' }}>{item.model} — {item.version}</Text>
          <Text>{item.checksum}</Text>
        </View>
      )} />
    </View>
  );
}

const styles = StyleSheet.create({ row: { padding: 12, borderBottomWidth: 1, borderColor: '#eee' } });
```

---

## File: src/screens/UploadFirmware.tsx

```tsx
import React, { useState } from 'react';
import { View, Button, Text, Platform } from 'react-native';
import * as DocumentPicker from 'expo-document-picker';
import { presignUpload, apiPOST } from '../api';
import { Buffer } from 'buffer';

export default function UploadFirmware({ navigation }: any) {
  const [progress, setProgress] = useState(0);
  const [uploading, setUploading] = useState(false);

  async function pickAndUpload() {
    const doc = await DocumentPicker.getDocumentAsync({ copyToCacheDirectory: true });
    if (doc.type !== 'success') return;
    setUploading(true);
    try {
      const presign = await presignUpload(doc.name || 'firmware.bin', doc.mimeType || 'application/octet-stream', doc.size || 0);
      const uploadUrl = presign.upload_url;
      const fileUri = doc.uri;
      const blob = await (await fetch(fileUri)).blob();

      const xhr = new XMLHttpRequest();
      xhr.open('PUT', uploadUrl);
      xhr.setRequestHeader('Content-Type', doc.mimeType || 'application/octet-stream');
      xhr.upload.onprogress = (e) => { if (e.lengthComputable) setProgress(e.loaded / e.total); };
      xhr.onload = async () => {
        if (xhr.status >= 200 && xhr.status < 300) {
          // Register metadata (server should verify checksum)
          await apiPOST('/api/v1/firmwares', { version: 'vX.Y.Z', model: 'modelX', blob_url: presign.blob_url, checksum: 'sha256:TODO', size: doc.size, signed_by: 'ci-signing-key' });
          alert('Upload complete');
          navigation.goBack();
        } else {
          alert('Upload failed: ' + xhr.status);
        }
      };
      xhr.onerror = () => alert('Upload error');
      xhr.send(blob);
    } catch (e: any) { alert('Error: ' + (e.message || e)); }
    finally { setUploading(false); setProgress(0); }
  }

  return (
    <View style={{ padding: 16 }}>
      <Button title={uploading ? 'Uploading...' : 'Pick & Upload Firmware'} onPress={pickAndUpload} disabled={uploading} />
      <Text>Progress: {(progress * 100).toFixed(1)}%</Text>
    </View>
  );
}
```

---

## File: src/screens/CreateCampaign.tsx

```tsx
import React, { useEffect, useState } from 'react';
import { View, TextInput, Button, Text } from 'react-native';
import { apiGET, apiPOST } from '../api';

export default function CreateCampaign({ navigation }: any) {
  const [name, setName] = useState('');
  const [groups, setGroups] = useState<any[]>([]);
  const [firmwares, setFirmwares] = useState<any[]>([]);
  const [groupId, setGroupId] = useState<number | null>(null);
  const [firmwareId, setFirmwareId] = useState<number | null>(null);

  useEffect(() => {
    apiGET('/api/v1/groups').then(setGroups).catch(() => {});
    apiGET('/api/v1/firmwares').then(setFirmwares).catch(() => {});
  }, []);

  async function create() {
    const payload = { name, target_group_id: groupId, firmware_id: firmwareId, rollout_strategy: { strategy: 'percentage', phases: [{ percent: 5, wait_minutes: 60 }, { percent: 100, wait_minutes: 0 }] } };
    await apiPOST('/api/v1/campaigns', payload);
    navigation.goBack();
  }

  return (
    <View style={{ padding: 16 }}>
      <Text>Name</Text>
      <TextInput value={name} onChangeText={setName} style={{ borderWidth: 1, padding: 8, marginBottom: 12 }} />
      <Text>Target group ID (enter numeric)</Text>
      <TextInput keyboardType="numeric" onChangeText={(t) => setGroupId(Number(t))} style={{ borderWidth: 1, padding: 8, marginBottom: 12 }} />
      <Text>Firmware ID</Text>
      <TextInput keyboardType="numeric" onChangeText={(t) => setFirmwareId(Number(t))} style={{ borderWidth: 1, padding: 8, marginBottom: 12 }} />
      <Button title="Create Campaign" onPress={create} />
    </View>
  );
}
```

---

### How to run
1. Copy this canvas content into a `mobile/` folder (or download file contents from the canvas UI).  
2. From `mobile/`: `npm install` then `npx expo start`.  
3. Run on an emulator or device.  

---

If you want, I can now:
- Add the backend code into the canvas as a second document (Node/Express + Postgres).  
- Or expand this canvas with prettier UI (react-native-paper components) and offline sync logic.  

Tell me which next step you want and I will update or create the corresponding canvas document.
