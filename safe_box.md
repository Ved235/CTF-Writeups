# Safe Box (VishwaCTF 2025)

The main idea was to somehow either "open" the safe or breakthrough it to get the flag.

Step-1:\
Launched [Asset Ripper](https://assetripper.github.io/AssetRipper/) and loaded the game folder into it.

Step-2:\
Explored the extracted content and found the "level0" collection that contained interesting assets.

![image](https://github.com/user-attachments/assets/ffb5044a-aaf8-4cc4-8bcf-03074262ec18)

Step-3:\
Located a GameObject named "flag" in the hierarchy, but couldn't find any useful information in Asset Ripper GUI.

![image](https://github.com/user-attachments/assets/3ba14ae0-bd87-4bfd-b4bc-d0b7a867fa75)

Step-4:\
Exported the project as a Unity project using Asset Ripper and opened it in Unity.

Step-5:\
In the Unity scene, found the "flag" object nested inside a container.

![image](https://github.com/user-attachments/assets/1275ee25-852f-4c3b-9e79-a6e250b4eb8c)

Step-6:\
Simply selected and dragged the flag object out of the safe container to reveal the flag.

![image](https://github.com/user-attachments/assets/918ed1ab-9e15-435a-baa1-8bdd4fc0a877)
