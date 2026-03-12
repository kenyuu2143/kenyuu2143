import random
import time

class CyberPartner:
    def __init__(self, name, model_type="SLIME-v1"):
        self.name = name
        self.model_type = model_type
        self.version = 1.0
        # サイバーパンクらしいステータス項目
        self.stats = {
            "Memory_Sync": 50,  # 同期率（親密度）
            "Data_Density": 100, # 空腹度のようなもの
            "Logic_Buffer": 20,  # 知力・魔法
            "Firewall_Integrity": 30 # 防御
        }
        self.is_glitched = False

    def display_status(self):
        print(f"--- [ACCESSING DATA: {self.name}] ---")
        print(f"TYPE: {self.model_type} | VER: {self.version}")
        for key, value in self.stats.items():
            bar = "█" * (value // 10) + "░" * (10 - value // 10)
            print(f"{key:18} | {bar} {value}%")
        print("------------------------------------")

    def pulse_check(self):
        """型破りなランダムイベント（バグ演出）"""
        if random.random() < 0.1:
            self.is_glitched = True
            print("⚠️ WARNING: SYSTEM GLITCH DETECTED ⚠️")
            self.stats["Memory_Sync"] += random.randint(-5, 15)

# インスタンス生成
my_buddy = CyberPartner("アーク・プロトタイプ")
my_buddy.display_status()
my_buddy.pulse_check()
