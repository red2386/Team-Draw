package com.example.teampick;

import net.kyori.adventure.text.Component;
import net.kyori.adventure.text.format.NamedTextColor;
import net.kyori.adventure.text.serializer.plain.PlainTextComponentSerializer;
import org.bukkit.*;
import org.bukkit.enchantments.Enchantment;
import org.bukkit.entity.Player;
import org.bukkit.event.EventHandler;
import org.bukkit.event.Listener;
import org.bukkit.event.inventory.InventoryClickEvent;
import org.bukkit.event.player.PlayerJoinEvent;
import org.bukkit.inventory.Inventory;
import org.bukkit.inventory.ItemFlag;
import org.bukkit.inventory.ItemStack;
import org.bukkit.inventory.meta.ItemMeta;
import org.bukkit.inventory.meta.SkullMeta;
import org.bukkit.scheduler.BukkitRunnable;

import java.util.*;

public class UIPicker implements Listener {

    public static final String UI_TITLE = "팀 뽑기 (좌클릭: 회전/정지)";
    // 비용이 들도록 바뀌었으므로 라벨 수정: 배정 없음만 강조
    public static final String UI_TITLE_TEST = "테스트 팀 뽑기 (배정 없음)";
    private static final int CENTER_SLOT = 22;

    private final Main plugin;
    private final DataStore dataStore;
    private final LeaderManager leaderManager;
    private final SpinService spinService;
    private final TeamService teamService;

    private final Map<UUID, SpinTask> running = new HashMap<>();

    public UIPicker(Main plugin, DataStore ds, LeaderManager lm, SpinService ss, TeamService ts) {
        this.plugin = plugin; this.dataStore = ds; this.leaderManager = lm; this.spinService = ss; this.teamService = ts;
    }

    public void openFor(Player viewer) {
        Inventory inv = Bukkit.createInventory(viewer, 54, Component.text(UI_TITLE));
        inv.clear();
        inv.setItem(CENTER_SLOT, placeholder("클릭해서 회전 시작", NamedTextColor.GRAY));
        viewer.openInventory(inv);
    }

    public void openForTest(Player viewer) {
        Inventory inv = Bukkit.createInventory(viewer, 54, Component.text(UI_TITLE_TEST));
        inv.clear();
        inv.setItem(CENTER_SLOT, placeholder("클릭해서 회전 시작(테스트/비용 소모)", NamedTextColor.AQUA));
        viewer.openInventory(inv);
    }

    private ItemStack placeholder(String text, NamedTextColor color) {
        ItemStack paper = new ItemStack(Material.PAPER);
        ItemMeta m = paper.getItemMeta();
        m.displayName(Component.text(text, color));
        paper.setItemMeta(m); return paper;
    }

    private ItemStack makeHead(UUID uuid, String spinPrefix, boolean glow) {
        ItemStack head = new ItemStack(Material.PLAYER_HEAD);
        SkullMeta meta = (SkullMeta) head.getItemMeta();

        String base;
        if (uuid == null) base = "Steve";
        else {
            OfflinePlayer off = Bukkit.getOfflinePlayer(uuid);
            base = (off.getName() == null) ? uuid.toString() : off.getName();
            meta.setOwningPlayer(off);
        }

        String name = (spinPrefix == null ? "" : (spinPrefix + " ")) + base;
        meta.displayName(Component.text(name, NamedTextColor.AQUA));
        meta.lore(List.of(Component.text("좌클릭: 회전/정지", NamedTextColor.GRAY)));
        if (glow) { meta.addEnchant(Enchantment.UNBREAKING, 1, true); meta.addItemFlags(ItemFlag.HIDE_ENCHANTS); }

        head.setItemMeta(meta);
        return head;
    }

    @EventHandler
    public void onClick(InventoryClickEvent e) {
        if (!(e.getWhoClicked() instanceof Player p)) return;

        String title = PlainTextComponentSerializer.plainText().serialize(e.getView().title());
        boolean testMode;
        if (UI_TITLE_TEST.equals(title)) testMode = true;
        else if (UI_TITLE.equals(title)) testMode = false;
        else return;

        e.setCancelled(true);
        if (!leaderManager.isLeader(p)) {
            p.sendMessage(Component.text("리더만 회전을 시작/정지할 수 있어요.", NamedTextColor.RED));
            return;
        }
        if (e.getSlot() != CENTER_SLOT) return;

        UUID leaderId = p.getUniqueId();
        SpinTask task = running.get(leaderId);

        if (task == null) {
            // ★ 테스트 모드도 동일하게 비용 소모
            int cost = spinService.getCurrentCost(p);
            if (!spinService.hasEnoughDiamonds(p)) {
                p.sendMessage(Component.text("다이아가 부족해요! (필요: " + cost + "개)", NamedTextColor.RED));
                p.playSound(p.getLocation(), Sound.ENTITY_VILLAGER_NO, 1f, 1f);
                return;
            }
            if (!spinService.payDiamonds(p)) {
                p.sendMessage(Component.text("다이아 지불에 실패했어요.", NamedTextColor.RED));
                return;
            }
            p.updateInventory();
            p.sendMessage(Component.text(
                    (testMode ? "[테스트] " : "") + "다이아 " + cost + "개 소모. (다음 비용: " + (cost + 64) + "개)",
                    NamedTextColor.GRAY));

            // 후보 수집: 테스트는 스티브 포함, 일반은 필터링
            List<UUID> pool = testMode
                    ? spinService.getAllCandidatesForTest()
                    : spinService.getFilteredCandidatesForNormal();
            if (testMode) { pool = new ArrayList<>(pool); pool.add(null); }

            if (pool.size() < 2) {
                p.sendMessage(Component.text("후보가 2명 이상일 때 회전합니다.", NamedTextColor.RED));
                return;
            }

            task = new SpinTask(p, e.getInventory(), pool, testMode);
            running.put(leaderId, task);
            task.start();
            p.sendMessage(Component.text("회전을 시작했습니다. 다시 좌클릭하면 감속/정지합니다.", NamedTextColor.AQUA));
        } else {
            task.requestStop();
            p.sendMessage(Component.text("감속 중... (효과음과 함께 천천히 멈춰요)", NamedTextColor.YELLOW));
        }
    }

    @EventHandler
    public void onJoin(PlayerJoinEvent e) { spinService.ensureCandidate(e.getPlayer()); }

    /** 중앙 한 칸 회전 + 부드러운 감속 */
    private class SpinTask extends BukkitRunnable {
        private final Player leader;
        private final Inventory inv;
        private final boolean testMode;
        private final List<UUID> seq;

        private final String[] SPINNER = {"⠋","⠙","⠹","⠸","⠼","⠴","⠦","⠧","⠇","⠏"};
        private int spinFrame = 0;

        private boolean stopping = false;
        private int idx = 0;
        private int tickCounter = 0;

        private int stepDelay = 2;        // 시작 속도(틱)
        private final int maxDelay = 12;  // 완전 정지 직전
        private int stepsSinceStop = 0;

        private double pitch = 1.8;
        private final double minPitch = 0.6;

        SpinTask(Player leader, Inventory inv, List<UUID> candidates, boolean testMode) {
            this.leader = leader; this.inv = inv; this.testMode = testMode;
            this.seq = new ArrayList<>(candidates);
        }

        void start() { runTaskTimer(plugin, 0L, 1L); } // 1틱 고정
        void requestStop() { stopping = true; }

        @Override
        public void run() {
            if (!leaderManager.isLeader(leader)) { finalizeSpin(null); return; }
            if (seq.isEmpty()) { finalizeSpin(null); return; }

            tickCounter++;
            if (tickCounter % stepDelay != 0) return;

            UUID cur = seq.get(idx);
            idx = (idx + 1) % seq.size();

            String prefix = SPINNER[spinFrame];
            spinFrame = (spinFrame + 1) % SPINNER.length;
            inv.setItem(CENTER_SLOT, makeHead(cur, prefix, true));

            if (!stopping) {
                leader.playSound(leader.getLocation(), Sound.UI_BUTTON_CLICK, 0.5f, (float) pitch);
            } else {
                pitch = Math.max(minPitch, pitch - 0.07);
                leader.playSound(leader.getLocation(), Sound.BLOCK_NOTE_BLOCK_HAT, 0.9f, (float) pitch);
                stepsSinceStop++;
                if (stepsSinceStop % 2 == 0 && stepDelay < maxDelay) stepDelay++;
                if (stepDelay >= maxDelay && stepsSinceStop >= 6) {
                    UUID winner = getUuidFromHead(inv.getItem(CENTER_SLOT));
                    finalizeSpin(winner);
                }
            }
        }

        private void finalizeSpin(UUID winner) {
            try { cancel(); } catch (Exception ignored) {}
            running.remove(leader.getUniqueId());

            if (testMode) {
                // 테스트: 배정 없음(단, 비용은 이미 소모됨)
                String who = (winner == null) ? "Steve"
                        : (Bukkit.getOfflinePlayer(winner).getName() == null ? winner.toString()
                        : Bukkit.getOfflinePlayer(winner).getName());
                leader.sendMessage(Component.text("[테스트] 당첨(배정 없음): " + who, NamedTextColor.AQUA));
                leader.playSound(leader.getLocation(), Sound.UI_TOAST_CHALLENGE_COMPLETE, 1f, 1f);
                inv.setItem(CENTER_SLOT, makeHead(winner, null, false));
                return;
            }

            if (winner == null) {
                leader.sendMessage(Component.text("유효한 플레이어가 아니라서 무효 처리되었습니다. 후보를 확인하세요.", NamedTextColor.RED));
                inv.setItem(CENTER_SLOT, makeHead(null, null, false));
                return;
            }
            if (spinService.alreadyWonBefore(winner) || teamService.isAlreadyAssigned(winner)) {
                leader.sendMessage(Component.text("이미 당첨/배정된 플레이어라서 건너뛰었어요.", NamedTextColor.YELLOW));
                return;
            }

            spinService.markWinner(winner);
            teamService.assignToLeaderTeam(winner, leaderManager);
            spinService.removeCandidate(winner); // 일반 모드: 당첨자는 후보 제외

            Location to = leader.getLocation().clone().add(1, 0, 0);
            Player online = Bukkit.getPlayer(winner);
            if (online != null) {
                online.teleportAsync(to);
                online.sendMessage(Component.text("팀에 합류! 리더 옆으로 이동했습니다.", NamedTextColor.GOLD));
            } else {
                teamService.queueTeleportOnJoin(winner, to);
            }

            dataStore.saveAll(leaderManager, teamService, spinService);

            inv.setItem(CENTER_SLOT, makeHead(winner, null, false));
            String name = Bukkit.getOfflinePlayer(winner).getName();
            leader.sendMessage(Component.text((name == null ? winner.toString() : name) + " 님 당첨! 팀 배정 완료.", NamedTextColor.GOLD));
            leader.playSound(leader.getLocation(), Sound.UI_TOAST_CHALLENGE_COMPLETE, 1f, 1f);
        }
    }

    private static UUID getUuidFromHead(ItemStack head) {
        if (head == null || head.getType() != Material.PLAYER_HEAD) return null;
        ItemMeta im = head.getItemMeta();
        if (!(im instanceof SkullMeta sm)) return null;
        OfflinePlayer op = sm.getOwningPlayer();
        return (op != null) ? op.getUniqueId() : null;
    }
}
