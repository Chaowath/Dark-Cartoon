# Dark-Cartoon
import React, { useState, useEffect } from 'react';
import {
  StyleSheet,
  Text,
  View,
  TextInput,
  TouchableOpacity,
  FlatList,
  Image,
  Alert,
  SafeAreaView,
  ScrollView,
  Modal
} from 'react-native';
import 'react-native-url-polyfill/auto';
import { createClient } from '@supabase/supabase-js';

// ==========================================
// กำหนดค่า SUPABASE URL และ KEY
// ==========================================
const SUPABASE_URL = 'https://nsqxmvgticnddnqnofzm.supabase.co';
const SUPABASE_ANON_KEY = 'sb_publishable_zyNFzy0vQ1CQlX0hqVUEbg_ACKjw3cJ';
const supabase = createClient(SUPABASE_URL, SUPABASE_ANON_KEY);

export default function App() {
  const [session, setSession] = useState(null);
  const [userRole, setUserRole] = useState('customer'); // 'customer' หรือ 'staff'
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [isRegistering, setIsRegistering] = useState(false);
  const [selectedRoleToRegister, setSelectedRoleToRegister] = useState('customer');

  // ข้อมูลการ์ตูน
  const [comics, setComics] = useState([]);
  const [searchQuery, setSearchQuery] = useState('');
  const [myList, setMyList] = useState([]);
  const [activeTab, setActiveTab] = useState('home'); // 'home', 'myList', 'staff'

  // ฟอร์มสำหรับพนักงาน (เพิ่มการ์ตูน)
  const [newTitle, setNewTitle] = useState('');
  const [newCover, setNewCover] = useState('');
  const [newDesc, setNewDesc] = useState('');

  // ฟอร์มสำหรับพนักงาน (เพิ่มตอน / Episodes)
  const [selectedComicForEpisode, setSelectedComicForEpisode] = useState(null);
  const [episodes, setEpisodes] = useState([]);
  const [epNumber, setEpNumber] = useState('');
  const [epTitle, setEpTitle] = useState('');
  const [epContent, setEpContent] = useState(''); // รองรับข้อความ, HTML หรือ URL รูปภาพหน้าเว็บ

  // อ่านเนื้อหาการ์ตูน & เลือกตอนอ่าน
  const [selectedComic, setSelectedComic] = useState(null);
  const [comicEpisodes, setComicEpisodes] = useState([]);
  const [readingEpisode, setReadingEpisode] = useState(null);

  useEffect(() => {
    supabase.auth.getSession().then(({ data: { session } }) => {
      setSession(session);
      if (session) fetchUserProfile(session.user.id);
    });

    const { data: authListener } = supabase.auth.onAuthStateChange((_event, session) => {
      setSession(session);
      if (session) {
        fetchUserProfile(session.user.id);
      } else {
        setUserRole('customer');
      }
    });

    return () => authListener.subscription.unsubscribe();
  }, []);

  useEffect(() => {
    fetchComics();
    if (session) {
      fetchMyList();
    }
  }, [session]);

  // ดึงข้อมูลประเภทผู้ใช้ (Role)
  const fetchUserProfile = async (userId) => {
    const { data } = await supabase
      .from('profiles')
      .select('role')
      .eq('id', userId)
      .single();

    if (data) {
      setUserRole(data.role);
    }
  };

  // ดึงข้อมูลรายการการ์ตูนทั้งหมด
  const fetchComics = async () => {
    const { data, error } = await supabase
      .from('comics')
      .select('*')
      .order('created_at', { ascending: false });

    if (!error && data) {
      setComics(data);
    }
  };

  // ดึงรายการการ์ตูนที่ลูกค้าบันทึกไว้
  const fetchMyList = async () => {
    if (!session) return;
    const { data, error } = await supabase
      .from('reading_list')
      .select('comic_id')
      .eq('user_id', session.user.id);

    if (!error && data) {
      setMyList(data.map((item) => item.comic_id));
    }
  };

  // ดึงข้อมูลตอนทั้งหมดของมังฮวาเรื่องนั้นๆ
  const fetchEpisodes = async (comicId) => {
    const { data, error } = await supabase
      .from('episodes')
      .select('*')
      .eq('comic_id', comicId)
      .order('episode_number', { ascending: true });

    if (!error && data) {
      setComicEpisodes(data);
    }
  };

  // ==========================================
  // ระบบเข้าสู่ระบบ / สมัครสมาชิก
  // ==========================================
  const handleAuth = async () => {
    if (!email || !password) {
      Alert.alert('กรุณากรอกข้อมูลให้ครบถ้วน');
      return;
    }

    if (isRegistering) {
      const { data, error } = await supabase.auth.signUp({ email, password });
      if (error) {
        Alert.alert('สมัครสมาชิกไม่สำเร็จ', error.message);
      } else if (data.user) {
        const staffEmails = ['staff1@darkcartoon.com', 'admin@darkcartoon.com']; 
        let finalRole = selectedRoleToRegister;

        if (staffEmails.includes(email.toLowerCase()) || email.toLowerCase().endsWith('@darkcartoon.com')) {
          finalRole = 'staff';
        }

        await supabase.from('profiles').insert([
          { id: data.user.id, email: email, role: finalRole }
        ]);

        Alert.alert('สมัครสมาชิกสำเร็จ!', `สิทธิ์การใช้งานของคุณคือ: ${finalRole}`);
      }
    } else {
      const { error } = await supabase.auth.signInWithPassword({ email, password });
      if (error) Alert.alert('เข้าสู่ระบบไม่สำเร็จ', error.message);
    }
  };

  const handleSignOut = () => {
    supabase.auth.signOut();
  };

  // เพิ่ม/ลบ รายการอ่านส่วนตัว (สำหรับลูกค้า)
  const toggleSaveComic = async (comicId) => {
    if (!session) return Alert.alert('กรุณาเข้าสู่ระบบก่อนใช้งาน');

    const isSaved = myList.includes(comicId);
    if (isSaved) {
      const { error } = await supabase
        .from('reading_list')
        .delete()
        .eq('user_id', session.user.id)
        .eq('comic_id', comicId);

      if (!error) setMyList(myList.filter((id) => id !== comicId));
    } else {
      const { error } = await supabase
        .from('reading_list')
        .insert([{ user_id: session.user.id, comic_id: comicId }]);

      if (!error) setMyList([...myList, comicId]);
    }
  };

  // เพิ่มการ์ตูนใหม่ (สำหรับพนักงาน)
  const handleAddComic = async () => {
    if (!newTitle || !newCover) {
      Alert.alert('กรุณากรอกชื่อเรื่องและลิงก์รูปปก');
      return;
    }

    const { error } = await supabase.from('comics').insert([
      { title: newTitle, cover_url: newCover, description: newDesc }
    ]);

    if (error) {
      Alert.alert('เกิดข้อผิดพลาดในการเพิ่มการ์ตูน', error.message);
    } else {
      Alert.alert('เพิ่มการ์ตูนเรียบร้อย');
      setNewTitle('');
      setNewCover('');
      setNewDesc('');
      fetchComics();
    }
  };

  // ลบการ์ตูน (สำหรับพนักงาน)
  const handleDeleteComic = async (id) => {
    const { error } = await supabase.from('comics').delete().eq('id', id);
    if (error) {
      Alert.alert('ไม่สามารถลบได้', error.message);
    } else {
      Alert.alert('ลบการ์ตูนเรียบร้อย');
      fetchComics();
    }
  };

  // เพิ่มตอน (Episode) สำหรับพนักงาน
  const handleAddEpisode = async () => {
    if (!selectedComicForEpisode || !epNumber || !epContent) {
      Alert.alert('กรุณากรอกเลขตอน และเนื้อหา/ลิงก์ HTML');
      return;
    }

    const { error } = await supabase.from('episodes').insert([
      {
        comic_id: selectedComicForEpisode.id,
        episode_number: parseInt(epNumber),
        title: epTitle || `ตอนที่ ${epNumber}`,
        content_html: epContent // สามารถใส่ URL รูปภาพ, เนื้อหา หรือ โค้ด HTML
      }
    ]);

    if (error) {
      Alert.alert('เพิ่มตอนไม่สำเร็จ', error.message);
    } else {
      Alert.alert('เพิ่มตอนสำเร็จ!');
      setEpNumber('');
      setEpTitle('');
      setEpContent('');
      fetchEpisodes(selectedComicForEpisode.id);
    }
  };

  // กรองการ์ตูนตามคำค้นหา
  const filteredComics = comics.filter((c) =>
    c.title.toLowerCase().includes(searchQuery.toLowerCase())
  );

  // การ์ตูนในรายการอ่านส่วนตัว
  const savedComicsList = comics.filter((c) => myList.includes(c.id));

  // หน้าจอเข้าสู่ระบบ / สมัครสมาชิก
  if (!session) {
    return (
      <SafeAreaView style={styles.container}>
        <View style={styles.authBox}>
          <Text style={styles.title}>DARK CARTOON</Text>
          <Text style={styles.subTitle}>{isRegistering ? 'สมัครสมาชิก' : 'เข้าสู่ระบบ'}</Text>

          <TextInput
            style={styles.input}
            placeholder="อีเมล"
            placeholderTextColor="#888"
            value={email}
            onChangeText={setEmail}
            autoCapitalize="none"
          />
          <TextInput
            style={styles.input}
            placeholder="รหัสผ่าน"
            placeholderTextColor="#888"
            secureTextEntry
            value={password}
            onChangeText={setPassword}
          />

          {isRegistering && (
            <View style={styles.rolePickerContainer}>
              <Text style={{ color: '#fff', marginBottom: 5 }}>สถานะผู้ใช้งานเริ่มต้น:</Text>
              <View style={{ flexDirection: 'row', gap: 10 }}>
                <TouchableOpacity
                  style={[styles.roleBtn, selectedRoleToRegister === 'customer' && styles.roleBtnActive]}
                  onPress={() => setSelectedRoleToRegister('customer')}
                >
                  <Text style={styles.btnText}>ลูกค้า</Text>
                </TouchableOpacity>
                <TouchableOpacity
                  style={[styles.roleBtn, selectedRoleToRegister === 'staff' && styles.roleBtnActive]}
                  onPress={() => setSelectedRoleToRegister('staff')}
                >
                  <Text style={styles.btnText}>พนักงาน</Text>
                </TouchableOpacity>
              </View>
            </View>
          )}

          <TouchableOpacity style={styles.primaryBtn} onPress={handleAuth}>
            <Text style={styles.primaryBtnText}>{isRegistering ? 'ยืนยันสมัครสมาชิก' : 'เข้าสู่ระบบ'}</Text>
          </TouchableOpacity>

          <TouchableOpacity onPress={() => setIsRegistering(!isRegistering)}>
            <Text style={styles.switchAuthText}>
              {isRegistering ? 'มีบัญชีแล้ว? เข้าสู่ระบบ' : 'ยังไม่มีบัญชี? สมัครสมาชิก'}
            </Text>
          </TouchableOpacity>
        </View>
      </SafeAreaView>
    );
  }

  // หน้าจอหลักเมื่อเข้าสู่ระบบสำเร็จ
  return (
    <SafeAreaView style={styles.container}>
      {/* Header */}
      <View style={styles.header}>
        <Text style={styles.headerTitle}>DARK CARTOON</Text>
        <TouchableOpacity onPress={handleSignOut}>
          <Text style={{ color: '#ff4444' }}>ออกจากระบบ</Text>
        </TouchableOpacity>
      </View>

      {/* Navigation Tabs */}
      <View style={styles.navBar}>
        <TouchableOpacity style={styles.navItem} onPress={() => setActiveTab('home')}>
          <Text style={[styles.navText, activeTab === 'home' && styles.activeNavText]}>หน้าแรก</Text>
        </TouchableOpacity>
        {userRole === 'customer' && (
          <TouchableOpacity style={styles.navItem} onPress={() => setActiveTab('myList')}>
            <Text style={[styles.navText, activeTab === 'myList' && styles.activeNavText]}>รายการของฉัน</Text>
          </TouchableOpacity>
        )}
        {userRole === 'staff' && (
          <TouchableOpacity style={styles.navItem} onPress={() => setActiveTab('staff')}>
            <Text style={[styles.navText, activeTab === 'staff' && styles.activeNavText]}>จัดการระบบ (พนักงาน)</Text>
          </TouchableOpacity>
        )}
      </View>

      {/* Tab: หน้าแรก (ค้นหา / ดูการ์ตูน / บันทึก) */}
      {activeTab === 'home' && (
        <View style={{ flex: 1, padding: 10 }}>
          <TextInput
            style={styles.searchInput}
            placeholder="🔍 ค้นหามังฮวา..."
            placeholderTextColor="#888"
            value={searchQuery}
            onChangeText={setSearchQuery}
          />
          <FlatList
            data={filteredComics}
            keyExtractor={(item) => item.id}
            numColumns={2}
            renderItem={({ item }) => (
              <TouchableOpacity style={styles.card} onPress={() => {
                setSelectedComic(item);
                fetchEpisodes(item.id);
              }}>
                <Image source={{ uri: item.cover_url || 'https://via.placeholder.com/150' }} style={styles.cardCover} />
                <Text style={styles.cardTitle} numberOfLines={1}>{item.title}</Text>
                {userRole === 'customer' && (
                  <TouchableOpacity
                    style={[styles.saveBtn, myList.includes(item.id) && styles.savedBtn]}
                    onPress={() => toggleSaveComic(item.id)}
                  >
                    <Text style={styles.btnText}>
                      {myList.includes(item.id) ? '✓ ในรายการ' : '+ เพิ่มในรายการ'}
                    </Text>
                  </TouchableOpacity>
                )}
              </TouchableOpacity>
            )}
          />
        </View>
      )}

      {/* Tab: รายการการ์ตูนของลูกค้า */}
      {activeTab === 'myList' && (
        <View style={{ flex: 1, padding: 10 }}>
          <Text style={styles.sectionHeader}>รายการมังฮวาที่บันทึกไว้</Text>
          <FlatList
            data={savedComicsList}
            keyExtractor={(item) => item.id}
            numColumns={2}
            renderItem={({ item }) => (
              <TouchableOpacity style={styles.card} onPress={() => {
                setSelectedComic(item);
                fetchEpisodes(item.id);
              }}>
                <Image source={{ uri: item.cover_url }} style={styles.cardCover} />
                <Text style={styles.cardTitle} numberOfLines={1}>{item.title}</Text>
                <TouchableOpacity style={styles.deleteBtn} onPress={() => toggleSaveComic(item.id)}>
                  <Text style={styles.btnText}>ลบออก</Text>
                </TouchableOpacity>
              </TouchableOpacity>
            )}
          />
        </View>
      )}

      {/* Tab: พนักงาน (เพิ่ม/ลบ การ์ตูน และ เพิ่มตอน) */}
      {activeTab === 'staff' && userRole === 'staff' && (
        <ScrollView style={{ flex: 1, padding: 15 }}>
          <Text style={styles.sectionHeader}>เพิ่มมังฮวาเรื่องใหม่</Text>
          <TextInput
            style={styles.input}
            placeholder="ชื่อเรื่องมังฮวา"
            placeholderTextColor="#888"
            value={newTitle}
            onChangeText={setNewTitle}
          />
          <TextInput
            style={styles.input}
            placeholder="URL รูปภาพปก (Image URL)"
            placeholderTextColor="#888"
            value={newCover}
            onChangeText={setNewCover}
          />
          <TextInput
            style={[styles.input, { height: 80 }]}
            placeholder="รายละเอียด / เรื่องย่อ"
            placeholderTextColor="#888"
            multiline
            value={newDesc}
            onChangeText={setNewDesc}
          />
          <TouchableOpacity style={styles.primaryBtn} onPress={handleAddComic}>
            <Text style={styles.primaryBtnText}>เพิ่มข้อมูลลงระบบ</Text>
          </TouchableOpacity>

          {/* ส่วนเพิ่มตอนของการ์ตูน */}
          <Text style={[styles.sectionHeader, { marginTop: 30 }]}>เพิ่มตอน (Episode)</Text>
          <Text style={{ color: '#aaa', marginBottom: 10 }}>เลือกเรื่องที่ต้องการเพิ่มตอน:</Text>
          <ScrollView horizontal style={{ marginBottom: 15 }}>
            {comics.map((c) => (
              <TouchableOpacity
                key={c.id}
                style={[styles.selectComicChip, selectedComicForEpisode?.id === c.id && styles.selectComicChipActive]}
                onPress={() => {
                  setSelectedComicForEpisode(c);
                  fetchEpisodes(c.id);
                }}
              >
                <Text style={{ color: '#fff' }}>{c.title}</Text>
              </TouchableOpacity>
            ))}
          </ScrollView>

          {selectedComicForEpisode && (
            <View style={{ backgroundColor: '#1e1e1e', padding: 12, borderRadius: 8, marginBottom: 20 }}>
              <Text style={{ color: '#e50914', fontWeight: 'bold', marginBottom: 10 }}>กำลังเพิ่มตอนให้: {selectedComicForEpisode.title}</Text>
              <TextInput
                style={styles.input}
                placeholder="เลขตอน (เช่น 1, 2, 3)"
                placeholderTextColor="#888"
                keyboardType="numeric"
                value={epNumber}
                onChangeText={setEpNumber}
              />
              <TextInput
                style={styles.input}
                placeholder="ชื่อตอน (ไม่ใส่ก็ได้ ระบบจะตั้งให้อัตโนมัติ)"
                placeholderTextColor="#888"
                value={epTitle}
                onChangeText={setEpTitle}
              />
              <TextInput
                style={[styles.input, { height: 100 }]}
                placeholder="เนื้อหา HTML / URL รูปภาพหน้าเว็บตอนนั้นๆ (คัดลอก HTML หรือลิงก์รูปภาพมาวาง)"
                placeholderTextColor="#888"
                multiline
                value={epContent}
                onChangeText={setEpContent}
              />
              <TouchableOpacity style={styles.primaryBtn} onPress={handleAddEpisode}>
                <Text style={styles.primaryBtnText}>บันทึกตอนใหม่</Text>
              </TouchableOpacity>
            </View>
          )}

          <Text style={[styles.sectionHeader, { marginTop: 10 }]}>รายการการ์ตูนในระบบ (พนักงานลบได้)</Text>
          {comics.map((item) => (
            <View key={item.id} style={styles.staffListItem}>
              <Text style={{ color: '#fff', flex: 1 }}>{item.title}</Text>
              <TouchableOpacity style={styles.deleteBtn} onPress={() => handleDeleteComic(item.id)}>
                <Text style={styles.btnText}>ลบการ์ตูน</Text>
              </TouchableOpacity>
            </View>
          ))}
        </ScrollView>
      )}

      {/* Modal รายละเอียดการ์ตูนและเลือกตอนอ่าน */}
      <Modal visible={!!selectedComic} animationType="slide">
        <SafeAreaView style={[styles.container, { padding: 15 }]}>
          {selectedComic && (
            <ScrollView>
              <TouchableOpacity onPress={() => { setSelectedComic(null); setReadingEpisode(null); }} style={{ marginBottom: 15 }}>
                <Text style={{ color: '#e50914', fontSize: 18 }}>← ย้อนกลับ</Text>
              </TouchableOpacity>

              {readingEpisode ? (
                // หน้าจออ่านเนื้อหาของตอนนั้นๆ
                <View>
                  <TouchableOpacity onPress={() => setReadingEpisode(null)} style={{ marginBottom: 10 }}>
                    <Text style={{ color: '#aaa' }}>← กลับไปหน้าเลือกตอน</Text>
                  </TouchableOpacity>
                  <Text style={{ color: '#fff', fontSize: 20, fontWeight: 'bold', marginBottom: 15 }}>
                    {selectedComic.title} - {readingEpisode.title}
                  </Text>
                  
                  {/* แสดงเนื้อหา (หากเป็น URL รูปภาพ หรือ HTML) */}
                  <Text style={{ color: '#fff', backgroundColor: '#1e1e1e', padding: 15, borderRadius: 8, lineHeight: 22 }}>
                    {readingEpisode.content_html}
                  </Text>
                </View>
              ) : (
                // หน้าจอแสดงรายละเอียดเรื่องย่อและรายการตอน
                <View>
                  <Image source={{ uri: selectedComic.cover_url }} style={{ width: '100%', height: 350, borderRadius: 10, resizeMode: 'cover' }} />
                  <Text style={{ color: '#fff', fontSize: 24, fontWeight: 'bold', marginVertical: 10 }}>
                    {selectedComic.title}
                  </Text>
                  <Text style={{ color: '#ccc', fontSize: 16, lineHeight: 22, marginBottom: 20 }}>
                    {selectedComic.description || 'ไม่มีรายละเอียดเรื่องย่อ'}
                  </Text>

                  <Text style={styles.sectionHeader}>รายการตอนทั้งหมด</Text>
                  {comicEpisodes.length === 0 ? (
                    <Text style={{ color: '#888' }}>ยังไม่มีตอนในเรื่องนี้</Text>
                  ) : (
                    comicEpisodes.map((ep) => (
                      <TouchableOpacity
                        key={ep.id}
                        style={styles.episodeItem}
                        onPress={() => setReadingEpisode(ep)}
                      >
                        <Text style={{ color: '#fff', fontWeight: 'bold' }}>{ep.title}</Text>
                        <Text style={{ color: '#e50914' }}>อ่าน →</Text>
                      </TouchableOpacity>
                    ))
                  )}
                </View>
              )}
            </ScrollView>
          )}
        </SafeAreaView>
      </Modal>
    </SafeAreaView>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#121212' },
  authBox: { flex: 1, justifyContent: 'center', padding: 20 },
  title: { fontSize: 32, fontWeight: 'bold', color: '#e50914', textAlign: 'center', marginBottom: 10 },
  subTitle: { fontSize: 18, color: '#fff', textAlign: 'center', marginBottom: 20 },
  input: { backgroundColor: '#1e1e1e', color: '#fff', padding: 12, borderRadius: 8, marginBottom: 12 },
  searchInput: { backgroundColor: '#1e1e1e', color: '#fff', padding: 12, borderRadius: 8, marginBottom: 15 },
  primaryBtn: { backgroundColor: '#e50914', padding: 15, borderRadius: 8, alignItems: 'center', marginTop: 10 },
  primaryBtnText: { color: '#fff', fontWeight: 'bold', fontSize: 16 },
  switchAuthText: { color: '#aaa', textAlign: 'center', marginTop: 15 },
  rolePickerContainer: { marginBottom: 15 },
  roleBtn: { flex: 1, padding: 10, backgroundColor: '#222', borderRadius: 6, alignItems: 'center' },
  roleBtnActive: { backgroundColor: '#e50914' },
  header: { flexDirection: 'row', justifyContent: 'space-between', padding: 15, borderBottomWidth: 1, borderColor: '#222', alignItems: 'center' },
  headerTitle: { color: '#e50914', fontWeight: 'bold', fontSize: 20 },
  navBar: { flexDirection: 'row', backgroundColor: '#1e1e1e', paddingVertical: 10 },
  navItem: { flex: 1, alignItems: 'center' },
  navText: { color: '#888', fontWeight: 'bold' },
  activeNavText: { color: '#e50914' },
  card: { flex: 0.5, margin: 5, backgroundColor: '#1e1e1e', borderRadius: 8, padding: 8 },
  cardCover: { width: '100%', height: 180, borderRadius: 6 },
  cardTitle: { color: '#fff', fontWeight: 'bold', marginTop: 8 },
  saveBtn: { backgroundColor: '#333', padding: 6, borderRadius: 4, marginTop: 6, alignItems: 'center' },
  savedBtn: { backgroundColor: '#2e7d32' },
  deleteBtn: { backgroundColor: '#c62828', padding: 6, borderRadius: 4, marginTop: 6, alignItems: 'center' },
  btnText: { color: '#fff', fontSize: 12 },
  sectionHeader: { color: '#fff', fontSize: 18, fontWeight: 'bold', marginBottom: 10 },
  staffListItem: { flexDirection: 'row', alignItems: 'center', backgroundColor: '#1e1e1e', padding: 10, borderRadius: 6, marginBottom: 8 },
  selectComicChip: { padding: 10, backgroundColor: '#222', borderRadius: 6, marginRight: 8, borderWidth: 1, borderColor: '#444' },
  selectComicChipActive: { borderColor: '#e50914', backgroundColor: '#1e1e1e' },
  episodeItem: { flexDirection: 'row', justifyContent: 'space-between', alignItems: 'center', backgroundColor: '#1e1e1e', padding: 15, borderRadius: 8, marginBottom: 8 }
});
