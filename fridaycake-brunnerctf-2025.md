# FridayCake (BrunnerCTF 2025)

> It's Friday - your family cake night! 🍰 But when you try to order your favorite cake, you find that someone has changed the app and locked the ordering screen!
>
> Recover the Secret Access Code and save the family's cake tradition.

In this challenge we were given an apk file. Following the standard procedure, I opened it in **jadx** and browsed through the source code. Find `MainActivity` and inside it we spot this:

```java
public static final void onCreate$lambda$0(EditText editText, MainActivity mainActivity, View view) {
        if (Authenticator.INSTANCE.checkCode(editText.getText().toString())) {
            Toast.makeText(mainActivity, "✅ Cake unlocked! Enjoy your Friday! 🎂", 1).show();
        } else {
            Toast.makeText(mainActivity, "❌ Wrong code! No cake for you...", 0).show();
        }
    }
```

Here a `checkCode()` belongs to the **Authenticator** class and on inspecting this new class we find:

```java
public final class Authenticator {
    public static final Authenticator INSTANCE = new Authenticator();

    private Authenticator() {
    }

    public final boolean checkCode(String input) {
        Intrinsics.checkNotNullParameter(input, "input");
        String str = StringsKt.reversed((CharSequence) input).toString() + "::CAKE::";
        ArrayList arrayList = new ArrayList(str.length());
        for (int i = 0; i < str.length(); i++) {
            arrayList.add(Character.valueOf((char) (str.charAt(i) + 2)));
        }
        return NativeChecker.INSTANCE.verifyCode(CollectionsKt.joinToString$default(arrayList, "", null, null, 0, null, null, 62, null));
    }
}
```

Over here there are some transformations being applied to the input string but the most useful part is the `verifyCode()` which belongs to **NativeChecker**. On opening the contents of this class it is visible that it loads and uses native functions from `libnative-lib.so` . Going into the resources folder in jadx we can export this library for the correct archiecture (I will be using x86-64).

```java
public final class NativeChecker {
    public static final NativeChecker INSTANCE = new NativeChecker();

    public final native String getDecryptedFlag();

    public final native boolean verifyCode(String code);

    private NativeChecker() {
    }

    static {
        System.loadLibrary("native-lib");
    }
}
```

Using Ghidra to decompile the code there are two main functions present as we expected `verifyCode()` and `getDecryptedFlag()` . The code verification function is decently long but asking ChatGPT to analyse it we get to know the function basically works by decoding a stored 56 byte secret, through a series of transformations converts it into the flag and stores it in a global variable and then compares our input against the 56 byte blob. The get flag function simply retrieves the decrypted flag from the global variable and returns it.&#x20;

Interesting, this means that if we triggering the `verifyCode` function always stores the decrypted flag as the validation is done afterwards. So to solve this challenge first trigger the verification and then trigger  `getDecryptedFlag` . I wrote a simple **frida** (it is a tool used for analysis of mobile apps) script:

```javascript
// frida -U -l hook.js -n FridayCake
Java.perform(function () {
  // Add a small timeout to allow everything to load
  setTimeout(function () {
    const NC = Java.use('dk.brunnerctf.fridaycake.NativeChecker');
    // 1) Trigger the native decrypt+store, supply any input
    NC.INSTANCE.value.verifyCode("x");
    // 2) Read the flag from the global
    const flag = NC.INSTANCE.value.getDecryptedFlag();
    console.log("FLAG:", flag);
  }, 500);
});
```

Open the app on your android emulator with frida setup and then run the script with the command `frida -U -l hook.js -n FridayCake` and from this we get our flag:

```bash
FLAG: brunner{Y0u_Us3d_Fr1d4_F0r_Gr4bb1ng_Th1s_R1ght?}
```
