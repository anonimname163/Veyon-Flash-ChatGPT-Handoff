# ФАКТИЧЕСКИЕ DIFF МОДИФИЦИРОВАННЫХ C++ И СБОРОЧНЫХ ФАЙЛОВ VEYON

**Базовый апстрим-коммит:** `1cebf45f63d03893daecf0393fe37cf1b29a2858` (Veyon 4.9.6)  
**Текущий коммит форка:** `165cf11e2c36e7befaec42be0a0533f9795a0667`  
**Репозиторий:** `C:\Users\Lolo\veyon` (ветка `main`, форк `anonimname163/veyon`)  
*Все чувствительные пароли и токены в diff заменены на `[REDACTED]`.*

---

## 1. `plugins/platform/windows/WindowsUserFunctions.cpp`

### Анализ изменений:
1. **Устранение бага в `prepareLogon()`:** В апстриме функция возвращала `true` и вызывала `logoff()`, когда никто не был залогинен (`!isAnyUserLoggedOn()`). Но без активного пользователя сессия не сбрасывалась, событие `applicationLoaded` никогда не наступало, и вход не происходил. В форке добавлен немедленный прямой вызов `performLogon()` при пустой сессии.
2. **Очистка поля и сброс шторки в `performLogon()`:** Убран ввод имени пользователя `username` и лишний `Tab`, так как в Windows 10/11 поле ввода пароля активно сразу. Добавлен сброс шторки блокировки (`XK_space`, `XK_Escape`), переключение раскладки на английский (`Alt_L + Shift_L`), очистка поля (`Ctrl+A` $\rightarrow$ `BackSpace`) и ввод чистого пароля.

### Diff:
```diff
diff --git a/plugins/platform/windows/WindowsUserFunctions.cpp b/plugins/platform/windows/WindowsUserFunctions.cpp
index 2dd6d3a20..9e2b765cd 100644
--- a/plugins/platform/windows/WindowsUserFunctions.cpp
+++ b/plugins/platform/windows/WindowsUserFunctions.cpp
@@ -194,19 +194,21 @@ bool WindowsUserFunctions::isAnyUserLoggedOn()
 
 bool WindowsUserFunctions::prepareLogon( const QString& username, const Password& password )
 {
-	if( m_logonHelper.prepare( username, password ) )
+	if( isAnyUserLoggedOn() )
 	{
+		m_logonHelper.prepare( username, password );
 		logoff();
 		return true;
 	}
 
-	return false;
+	return performLogon( username, password );
 }
 
 
 
 bool WindowsUserFunctions::performLogon( const QString& username, const Password& password )
 {
+	Q_UNUSED(username)
 	WindowsPlatformConfiguration config( &VeyonCore::config() );
 
 	DesktopInputController input( config.logonKeyPressInterval() );
@@ -219,13 +221,6 @@ bool WindowsUserFunctions::performLogon( const QString& username, const Password
 		}
 	};
 
-	const auto sendString = [&input]( const QString& string ) {
-		for( const auto& character : string )
-		{
-			input.pressAndReleaseKey( character );
-		}
-	};
-
 	WindowsInputDeviceFunctions::stopOnScreenKeyboard();
 
 	ctrlAltDel();
@@ -238,13 +233,32 @@ bool WindowsUserFunctions::performLogon( const QString& username, const Password
 		QThread::msleep( static_cast<unsigned long>( config.logonInputStartDelay() ) );
 	}
 
-	input.pressAndReleaseKey( XK_Delete );
+	// Wake up / dismiss curtain
+	input.pressAndReleaseKey( XK_space );
+	QThread::msleep( 300 );
+	input.pressAndReleaseKey( XK_Escape );
+	QThread::msleep( 200 );
 
-	sendString( username );
+	// Switch layout to English (Left Alt + Left Shift)
+	input.pressKey( XK_Alt_L );
+	input.pressAndReleaseKey( XK_Shift_L );
+	input.releaseKey( XK_Alt_L );
+	QThread::msleep( 200 );
 
-	input.pressAndReleaseKey( XK_Tab );
+	// Clear field (Ctrl + A, BackSpace)
+	input.pressKey( XK_Control_L );
+	input.pressAndReleaseKey( 'a' );
+	input.releaseKey( XK_Control_L );
+	QThread::msleep( 50 );
+	input.pressAndReleaseKey( XK_BackSpace );
+	QThread::msleep( 50 );
 
-	sendString( QString::fromUtf8( password.toByteArray() ) );
+	// Type password
+	const auto plainPassword = QString::fromUtf8( password.toByteArray() );
+	for( const auto& character : plainPassword )
+	{
+		input.pressAndReleaseKey( character );
+	}
 
 	input.pressAndReleaseKey( XK_Return );
```

---

## 2. `plugins/usersessioncontrol/UserSessionControlPlugin.cpp`

### Анализ изменений:
1. **Удаление модального окна в GUI мастера:** Для фичи `m_userLoginFeature` убран вызов интерактивного диалога `UserLoginDialog`. Данные читаются автоматически из `QSettings("Veyon", "VeyonMaster")`.
2. **Добавление фичи `m_setPasswordFeature`:** Создан пункт меню и диалог для централизованной смены пароля пользователя ученика.
3. **Обработка на клиенте (`handleFeatureMessage`):**
   - Для `m_setPasswordFeature`: вызов `net user <username> <pass>`. Если пользователя нет, выполняется `/add` и добавление в группы «Пользователи» / «Users».
   - Для `m_userLoginFeature`: вызов `reg add` для системного автологина и раскладки `00000409`. Запуск отдельного потока с `OpenInputDesktop()`, `SetThreadDesktop()`, сбросом шторки и отправкой Unicode-символов пароля через `KEYEVENTF_UNICODE`.

### Diff:
```diff
diff --git a/plugins/usersessioncontrol/UserSessionControlPlugin.cpp b/plugins/usersessioncontrol/UserSessionControlPlugin.cpp
index 73a388ab7..728b61c92 100644
--- a/plugins/usersessioncontrol/UserSessionControlPlugin.cpp
+++ b/plugins/usersessioncontrol/UserSessionControlPlugin.cpp
@@ -21,11 +21,21 @@
 #include "UserSessionControlPlugin.h"
 #include "FeatureMessage.h"
 #include "VeyonCore.h"
+#include "VeyonServerInterface.h"
 
 #include <QCoreApplication>
+#include <QDialog>
+#include <QDialogButtonBox>
+#include <QFormLayout>
+#include <QLineEdit>
+#include <QProcess>
+#include <QSettings>
+#include <thread>
 
 #ifdef Q_OS_WIN
 #include <windows.h>
+#include <winuser.h>
+#include <winnls.h>
 #endif
 
 UserSessionControlPlugin::UserSessionControlPlugin( QObject* parent ) :
@@ -34,6 +44,11 @@ UserSessionControlPlugin::UserSessionControlPlugin( QObject* parent ) :
 	m_userLogoffFeature( Feature::Type::Action,
 						 Feature::Uid( "{34559b38-e6b7-4a00-ab86-419b7835843a}" ),
 						 createFeatureWorker<UserLogoffFeatureWorker>() ),
+	m_setPasswordFeature( Feature::Type::Action,
+						  Feature::Uid( "{a1b2c3d4-e5f6-7a8b-9c0d-1e2f3a4b5c6d}" ),
+						  Feature::WorkerFactory(),
+						  Feature::Flags( Feature::Flag::ComputerControlInterfaceFeature |
+										  Feature::Flag::MasterFeature ) ),
 	m_userFeatures( { m_userLoginFeature,
 					  m_userLogoffFeature } )
 {
@@ -52,6 +67,11 @@ UserSessionControlPlugin::UserSessionControlPlugin( QObject* parent ) :
 	m_userLogoffFeature.setIconPath( QStringLiteral(":/usersessioncontrol/user-logoff.png") );
 	m_userLogoffFeature.setDescription( tr( "Log off the currently logged on user" ) );
 
+	m_setPasswordFeature.setName( tr( "Set user password" ) );
+	m_setPasswordFeature.setDisplayName( tr( "Set password" ) );
+	m_setPasswordFeature.setIconPath( QStringLiteral(":/usersessioncontrol/user-login.png") );
+	m_setPasswordFeature.setDescription( tr( "Set password for user on selected computers" ) );
+
 	for( auto& feature : m_userFeatures )
 	{
 		feature.setPluginUid( uid() );
@@ -88,14 +108,53 @@ bool UserSessionControlPlugin::startFeature( const Feature& feature,
 {
 	if( feature == m_userLoginFeature )
 	{
-		UserLoginDialog loginDialog;
-		if( loginDialog.exec() == QDialog::Accepted )
+		QSettings settings( QStringLiteral("Veyon"), QStringLiteral("VeyonMaster") );
+		const auto username = settings.value( QStringLiteral("AutoLogin/Username"), QStringLiteral("Student") ).toString();
+		const auto password = settings.value( QStringLiteral("AutoLogin/Password"), QStringLiteral("[REDACTED_PASSWORD]") ).toByteArray();
+
+		return controlFeature( feature.uid(), Operation::Start,
+							   {
+								   { argToString(Argument::Username), username },
+								   { argToString(Argument::Password), password }
+							   },
+							   computerControlInterfaces );
+	}
+	else if( feature == m_setPasswordFeature )
+	{
+		QSettings settings( QStringLiteral("Veyon"), QStringLiteral("VeyonMaster") );
+		const auto currentUsername = settings.value( QStringLiteral("AutoLogin/Username"), QStringLiteral("Student") ).toString();
+
+		QDialog dialog;
+		dialog.setWindowTitle( tr( "Set User Password" ) );
+		auto formLayout = new QFormLayout( &dialog );
+
+		auto usernameEdit = new QLineEdit( currentUsername, &dialog );
+		auto passwordEdit = new QLineEdit( &dialog );
+		passwordEdit->setEchoMode( QLineEdit::Password );
+
+		formLayout->addRow( tr( "Username:" ), usernameEdit );
+		formLayout->addRow( tr( "New password:" ), passwordEdit );
+
+		auto buttonBox = new QDialogButtonBox( QDialogButtonBox::Ok | QDialogButtonBox::Cancel, &dialog );
+		QObject::connect( buttonBox, &QDialogButtonBox::accepted, &dialog, &QDialog::accept );
+		QObject::connect( buttonBox, &QDialogButtonBox::rejected, &dialog, &QDialog::reject );
+		formLayout->addWidget( buttonBox );
+
+		if( dialog.exec() == QDialog::Accepted && !usernameEdit->text().trimmed().isEmpty() )
 		{
+			const auto newUsername = usernameEdit->text().trimmed();
+			const auto newPassword = passwordEdit->text().toUtf8();
+
+			settings.setValue( QStringLiteral("AutoLogin/Username"), newUsername );
+			settings.setValue( QStringLiteral("AutoLogin/Password"), newPassword );
+
 			return controlFeature( feature.uid(), Operation::Start,
 								   {
-									   { argToString(Argument::Username), loginDialog.username() },
-									   { argToString(Argument::Password), loginDialog.password().toByteArray() }
+									   { argToString(Argument::Username), newUsername },
+									   { argToString(Argument::Password), newPassword }
 								   },
 								   computerControlInterfaces );
 		}
+		return false;
 	}
 	else if( feature == m_userLogoffFeature )
 	{
@@ -136,10 +195,131 @@ bool UserSessionControlPlugin::handleFeatureMessage( VeyonServerInterface& serve
 
 	if( message.featureUid() == m_userLoginFeature.uid() )
 	{
-		VeyonCore::platform().userFunctions().prepareLogon( message.argument( Argument::Username ).toString(),
-															VeyonCore::cryptoCore().decryptPassword(
-																message.argument( Argument::Password ).toString() ) );
+		const auto username = message.argument( Argument::Username ).toString();
+		const auto password = VeyonCore::cryptoCore().decryptPassword(
+									message.argument( Argument::Password ).toString() );
+		const auto plainPassword = QString::fromUtf8( password.toByteArray() );
+
+#ifdef Q_OS_WIN
+		// 1. Ensure registry settings: English layout, no lock screen curtain, AutoAdminLogon
+		QProcess::execute( QStringLiteral("reg"), { QStringLiteral("add"), QStringLiteral("HKEY_USERS\\.DEFAULT\\Keyboard Layout\\Preload"), QStringLiteral("/v"), QStringLiteral("1"), QStringLiteral("/t"), QStringLiteral("REG_SZ"), QStringLiteral("/d"), QStringLiteral("00000409"), QStringLiteral("/f") } );
+		QProcess::execute( QStringLiteral("reg"), { QStringLiteral("add"), QStringLiteral("HKEY_USERS\\.DEFAULT\\Keyboard Layout\\Preload"), QStringLiteral("/v"), QStringLiteral("2"), QStringLiteral("/t"), QStringLiteral("REG_SZ"), QStringLiteral("/d"), QStringLiteral("00000419"), QStringLiteral("/f") } );
+		QProcess::execute( QStringLiteral("reg"), { QStringLiteral("add"), QStringLiteral("HKLM\\SOFTWARE\\Policies\\Microsoft\\Windows\\Personalization"), QStringLiteral("/v"), QStringLiteral("NoLockScreen"), QStringLiteral("/t"), QStringLiteral("REG_DWORD"), QStringLiteral("/d"), QStringLiteral("1"), QStringLiteral("/f") } );
+		QProcess::execute( QStringLiteral("reg"), { QStringLiteral("add"), QStringLiteral("HKLM\\SOFTWARE\\Microsoft\\Windows\\CurrentVersion\\Policies\\System"), QStringLiteral("/v"), QStringLiteral("dontdisplaylastusername"), QStringLiteral("/t"), QStringLiteral("REG_DWORD"), QStringLiteral("/d"), QStringLiteral("0"), QStringLiteral("/f") } );
+		QProcess::execute( QStringLiteral("reg"), { QStringLiteral("add"), QStringLiteral("HKLM\\SOFTWARE\\Microsoft\\Windows NT\\CurrentVersion\\Winlogon"), QStringLiteral("/v"), QStringLiteral("AutoAdminLogon"), QStringLiteral("/t"), QStringLiteral("REG_SZ"), QStringLiteral("/d"), QStringLiteral("1"), QStringLiteral("/f") } );
+		QProcess::execute( QStringLiteral("reg"), { QStringLiteral("add"), QStringLiteral("HKLM\\SOFTWARE\\Microsoft\\Windows NT\\CurrentVersion\\Winlogon"), QStringLiteral("/v"), QStringLiteral("DefaultUserName"), QStringLiteral("/t"), QStringLiteral("REG_SZ"), QStringLiteral("/d"), username, QStringLiteral("/f") } );
+		QProcess::execute( QStringLiteral("reg"), { QStringLiteral("add"), QStringLiteral("HKLM\\SOFTWARE\\Microsoft\\Windows NT\\CurrentVersion\\Winlogon"), QStringLiteral("/v"), QStringLiteral("DefaultPassword"), QStringLiteral("/t"), QStringLiteral("REG_SZ"), QStringLiteral("/d"), plainPassword, QStringLiteral("/f") } );
+
+		// 2. Perform automated logon sequence in a detached worker thread
+		std::thread logonThread( [plainPassword]() {
+			const auto sendUnicodeChar = []( wchar_t c ) {
+				INPUT inputs[2] = {};
+				inputs[0].type = INPUT_KEYBOARD;
+				inputs[0].ki.wScan = c;
+				inputs[0].ki.dwFlags = KEYEVENTF_UNICODE;
+				inputs[1].type = INPUT_KEYBOARD;
+				inputs[1].ki.wScan = c;
+				inputs[1].ki.dwFlags = KEYEVENTF_UNICODE | KEYEVENTF_KEYUP;
+				SendInput( 2, inputs, sizeof(INPUT) );
+				Sleep( 25 );
+			};
+
+			const auto sendVirtualKey = []( BYTE vk ) {
+				keybd_event( vk, 0, 0, 0 );
+				Sleep( 25 );
+				keybd_event( vk, 0, KEYEVENTF_KEYUP, 0 );
+				Sleep( 25 );
+			};
+
+			// Attach to the current input desktop
+			HDESK desktop = OpenInputDesktop( 0, FALSE, GENERIC_WRITE );
+			if( desktop )
+			{
+				SetThreadDesktop( desktop );
+			}
+
+			// Wake up / dismiss lock screen curtain (space / return)
+			sendVirtualKey( VK_SPACE );
+			Sleep( 400 );
+			sendVirtualKey( VK_RETURN );
+			Sleep( 400 );
+			sendVirtualKey( VK_ESCAPE );
+			Sleep( 200 );
+
+			// Switch keyboard layout to English (00000409)
+			HKL hklEng = LoadKeyboardLayoutW( L"00000409", KLF_ACTIVATE | KLF_REORDER );
+			if( hklEng )
+			{
+				ActivateKeyboardLayout( hklEng, KLF_SETFORPROCESS );
+				HWND hwndFore = GetForegroundWindow();
+				if( hwndFore )
+				{
+					PostMessageW( hwndFore, WM_INPUTLANGCHANGEREQUEST, 0, (LPARAM)hklEng );
+				}
+			}
+
+			// Also simulate Alt+Shift to toggle layout to English
+			keybd_event( VK_MENU, 0, 0, 0 );
+			keybd_event( VK_SHIFT, 0, 0, 0 );
+			Sleep( 30 );
+			keybd_event( VK_SHIFT, 0, KEYEVENTF_KEYUP, 0 );
+			keybd_event( VK_MENU, 0, KEYEVENTF_KEYUP, 0 );
+			Sleep( 200 );
+
+			// Clear password field: Ctrl+A then Backspace
+			keybd_event( VK_CONTROL, 0, 0, 0 );
+			keybd_event( 'A', 0, 0, 0 );
+			Sleep( 25 );
+			keybd_event( 'A', 0, KEYEVENTF_KEYUP, 0 );
+			keybd_event( VK_CONTROL, 0, KEYEVENTF_KEYUP, 0 );
+			Sleep( 50 );
+			sendVirtualKey( VK_BACK );
+			Sleep( 100 );
+
+			// Type the password characters via KEYEVENTF_UNICODE
+			for( const auto& ch : plainPassword )
+			{
+				sendUnicodeChar( ch.unicode() );
+			}
+			Sleep( 150 );
+
+			// Submit password: press Enter
+			sendVirtualKey( VK_RETURN );
+
+			if( desktop )
+			{
+				CloseDesktop( desktop );
+			}
+		} );
+		logonThread.detach();
+
 		return true;
+#else
+		VeyonCore::platform().userFunctions().prepareLogon( username, password );
+		return true;
+#endif
+	}
+	else if( message.featureUid() == m_setPasswordFeature.uid() )
+	{
+		const auto username = message.argument( Argument::Username ).toString();
+		const auto password = VeyonCore::cryptoCore().decryptPassword(
+									message.argument( Argument::Password ).toString() );
+		if( !username.isEmpty() )
+		{
+#ifdef Q_OS_WIN
+			const auto plainPassword = QString::fromUtf8( password.toByteArray() );
+			const auto exitCode = QProcess::execute( QStringLiteral("net"), { QStringLiteral("user"), username, plainPassword } );
+			if( exitCode != 0 )
+			{
+				vInfo() << "User does not exist, creating user:" << username;
+				QProcess::execute( QStringLiteral("net"), { QStringLiteral("user"), username, plainPassword, QStringLiteral("/add") } );
+				QProcess::execute( QStringLiteral("net"), { QStringLiteral("localgroup"), QStringLiteral("Пользователи"), username, QStringLiteral("/add") } );
+				QProcess::execute( QStringLiteral("net"), { QStringLiteral("localgroup"), QStringLiteral("Users"), username, QStringLiteral("/add") } );
+			}
+#endif
+			return true;
+		}
+		return false;
 	}
 	else if( message.featureUid() == m_userLogoffFeature.uid() )
 	{
```

---

## 3. `plugins/usersessioncontrol/UserSessionControlPlugin.h`

```diff
diff --git a/plugins/usersessioncontrol/UserSessionControlPlugin.h b/plugins/usersessioncontrol/UserSessionControlPlugin.h
index 8e4528ea6..c5bc8b108 100644
--- a/plugins/usersessioncontrol/UserSessionControlPlugin.h
+++ b/plugins/usersessioncontrol/UserSessionControlPlugin.h
@@ -53,6 +53,7 @@ private:
 
 	Feature m_userLoginFeature;
 	Feature m_userLogoffFeature;
+	Feature m_setPasswordFeature;
 
 	FeatureList m_userFeatures;
 
```

---

## 4. CMake и конфигурационные файлы
В `CMakeLists.txt` и `plugins/usersessioncontrol/CMakeLists.txt` были добавлены настройки для отключения жесткой привязки к минорной версии Qt (`QT_NO_VERSION_TAGGING`), отключения `-Werror` при сборке более новым MinGW GCC и согласования версии `_WIN32_WINNT=0x0601`.
