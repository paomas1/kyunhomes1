package com.kyunsmp.homes;

import net.kyori.adventure.text.Component;
import net.kyori.adventure.text.format.TextDecoration;
import net.kyori.adventure.text.serializer.legacy.LegacyComponentSerializer;
import org.bukkit.Bukkit;
import org.bukkit.Location;
import org.bukkit.Material;
import org.bukkit.OfflinePlayer;
import org.bukkit.command.Command;
import org.bukkit.command.CommandExecutor;
import org.bukkit.command.CommandSender;
import org.bukkit.command.TabCompleter;
import org.bukkit.configuration.ConfigurationSection;
import org.bukkit.configuration.file.YamlConfiguration;
import org.bukkit.entity.Player;
import org.bukkit.event.EventHandler;
import org.bukkit.event.Listener;
import org.bukkit.event.inventory.ClickType;
import org.bukkit.event.inventory.InventoryClickEvent;
import org.bukkit.event.player.PlayerMoveEvent;
import org.bukkit.event.player.PlayerQuitEvent;
import org.bukkit.inventory.Inventory;
import org.bukkit.inventory.InventoryHolder;
import org.bukkit.inventory.ItemStack;
import org.bukkit.inventory.meta.ItemMeta;
import org.bukkit.plugin.java.JavaPlugin;
import org.bukkit.scheduler.BukkitTask;

import java.io.File;
import java.io.IOException;
import java.util.ArrayList;
import java.util.Collections;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Objects;
import java.util.UUID;
import java.util.regex.Pattern;

public class KyunHomes extends JavaPlugin implements CommandExecutor, TabCompleter, Listener {

    private static final LegacyComponentSerializer LEGACY = LegacyComponentSerializer.legacyAmpersand();
    private static final Pattern NAME_PATTERN = Pattern.compile("^[a-z0-9_-]{1,16}$");
    private static final int HOME_SLOTS = 45; // menunun ev bolumu (5 satir)

    private File dataFile;
    private YamlConfiguration data;
    private final Map<UUID, BukkitTask> pendingTasks = new HashMap<>();
    private final Map<UUID, Location> pendingStart = new HashMap<>();

    // ------------------------------------------------------------------
    // Baslangic
    // ------------------------------------------------------------------

    @Override
    public void onEnable() {
        saveDefaultConfig();
        dataFile = new File(getDataFolder(), "data.yml");
        data = YamlConfiguration.loadConfiguration(dataFile);

        for (String c : List.of("sethome", "delhome", "home", "homelimit")) {
            if (getCommand(c) != null) {
                getCommand(c).setExecutor(this);
                getCommand(c).setTabCompleter(this);
            }
        }
        getServer().getPluginManager().registerEvents(this, this);
        getLogger().info("KyunHomes aktif.");
    }

    // ------------------------------------------------------------------
    // Yardimci metodlar
    // ------------------------------------------------------------------

    private void msg(CommandSender s, String text) {
        s.sendMessage(LEGACY.deserialize("&8[&bEv&8] &7" + text));
    }

    private String homesPath(UUID id) {
        return "homes." + id;
    }

    private List<String> getHomeNames(UUID id) {
        ConfigurationSection sec = data.getConfigurationSection(homesPath(id));
        if (sec == null) return new ArrayList<>();
        return new ArrayList<>(sec.getKeys(false));
    }

    private int getLimit(UUID id) {
        return data.getInt("limits." + id, getConfig().getInt("default-limit", 5));
    }

    private void saveData() {
        try {
            data.save(dataFile);
        } catch (IOException e) {
            getLogger().severe("data.yml kaydedilemedi: " + e.getMessage());
        }
    }

    private String freeAutoName(UUID id) {
        int n = 1;
        while (data.contains(homesPath(id) + ".ev" + n)) n++;
        return "ev" + n;
    }

    private void setHome(Player p, String name) {
        UUID id = p.getUniqueId();
        String path = homesPath(id) + "." + name;
        boolean exists = data.contains(path);
        int limit = getLimit(id);
        if (!exists && getHomeNames(id).size() >= limit) {
            msg(p, "&cEv limitine ulastin! (&f" + limit + "&c)");
            return;
        }
        data.set(path, p.getLocation());
        saveData();
        msg(p, exists ? "&a'" + name + "' evi yeni konumuna tasindi."
                      : "&a'" + name + "' evi kuruldu.");
    }

    private boolean deleteHome(UUID id, String name) {
        String path = homesPath(id) + "." + name;
        if (!data.contains(path)) return false;
        data.set(path, null);
        ConfigurationSection sec = data.getConfigurationSection(homesPath(id));
        if (sec != null && sec.getKeys(false).isEmpty()) {
            data.set(homesPath(id), null);
        }
        saveData();
        return true;
    }

    // ------------------------------------------------------------------
    // Isinlanma (bekleme + hareket edince iptal)
    // ------------------------------------------------------------------

    private void startTeleport(Player p, Location dest, String homeName) {
        UUID id = p.getUniqueId();
        cancelPending(id, false);
        int delay = getConfig().getInt("teleport-delay-seconds", 3);

        if (delay <= 0 || p.hasPermission("kyunhomes.bypassdelay")) {
            p.teleport(dest);
            msg(p, "&a'" + homeName + "' evine isinlandin.");
            return;
        }

        pendingStart.put(id, p.getLocation().clone());
        msg(p, "&e" + delay + " saniye icinde isinlanacaksin, hareket etme!");
        BukkitTask task = Bukkit.getScheduler().runTaskLater(this, () -> {
            pendingTasks.remove(id);
            pendingStart.remove(id);
            if (p.isOnline()) {
                p.teleport(dest);
                msg(p, "&a'" + homeName + "' evine isinlandin.");
            }
        }, delay * 20L);
        pendingTasks.put(id, task);
    }

    private void cancelPending(UUID id, boolean notify) {
        BukkitTask t = pendingTasks.remove(id);
        pendingStart.remove(id);
        if (t != null) {
            t.cancel();
            if (notify) {
                Player p = Bukkit.getPlayer(id);
                if (p != null) msg(p, "&cHareket ettin, isinlanma iptal edildi.");
            }
        }
    }

    @EventHandler
    public void onMove(PlayerMoveEvent e) {
        UUID id = e.getPlayer().getUniqueId();
        Location start = pendingStart.get(id);
        if (start == null) return;
        Location to = e.getTo();
        if (to == null) return;
        if (to.getBlockX() != start.getBlockX()
                || to.getBlockY() != start.getBlockY()
                || to.getBlockZ() != start.getBlockZ()
                || !Objects.equals(to.getWorld(), start.getWorld())) {
            cancelPending(id, true);
        }
    }

    @EventHandler
    public void onQuit(PlayerQuitEvent e) {
        cancelPending(e.getPlayer().getUniqueId(), false);
    }

    // ------------------------------------------------------------------
    // Komutlar
    // ------------------------------------------------------------------

    @Override
    public boolean onCommand(CommandSender sender, Command cmd, String label, String[] args) {
        String name = cmd.getName().toLowerCase();

        if (name.equals("homelimit")) {
            return handleLimit(sender, args);
        }

        if (!(sender instanceof Player p)) {
            msg(sender, "&cBu komut sadece oyuncular icindir.");
            return true;
        }

        switch (name) {
            case "sethome" -> {
                String hn;
                if (args.length == 0) {
                    hn = freeAutoName(p.getUniqueId());
                } else {
                    hn = args[0].toLowerCase();
                }
                if (!NAME_PATTERN.matcher(hn).matches()) {
                    msg(p, "&cGecersiz isim. Sadece harf, rakam, _ ve - kullan (en fazla 16 karakter).");
                    return true;
                }
                setHome(p, hn);
            }
            case "delhome" -> {
                if (args.length == 0) {
                    msg(p, "&cKullanim: &f/delhome <isim>");
                    return true;
                }
                String hn = args[0].toLowerCase();
                if (deleteHome(p.getUniqueId(), hn)) {
                    msg(p, "&a'" + hn + "' evi silindi.");
                } else {
                    msg(p, "&c'" + hn + "' adinda bir evin yok.");
                }
            }
            case "home" -> {
                if (args.length == 0) {
                    openMenu(p, 0);
                } else {
                    String hn = args[0].toLowerCase();
                    Location loc = data.getLocation(homesPath(p.getUniqueId()) + "." + hn);
                    if (loc == null || loc.getWorld() == null) {
                        msg(p, "&c'" + hn + "' adinda bir evin yok.");
                        return true;
                    }
                    startTeleport(p, loc, hn);
                }
            }
            default -> { }
        }
        return true;
    }

    private boolean handleLimit(CommandSender sender, String[] args) {
        if (!sender.hasPermission("kyunhomes.admin")) {
            msg(sender, "&cBunun icin yetkin yok.");
            return true;
        }
        if (args.length < 1) {
            msg(sender, "&cKullanim: &f/homelimit <oyuncu> [sayi]");
            return true;
        }
        Player online = Bukkit.getPlayerExact(args[0]);
        OfflinePlayer target = online != null ? online : Bukkit.getOfflinePlayerIfCached(args[0]);
        if (target == null) {
            msg(sender, "&cOyuncu bulunamadi (sunucuya daha once girmis olmali).");
            return true;
        }
        UUID id = target.getUniqueId();
        String tName = target.getName() != null ? target.getName() : args[0];

        if (args.length == 1) {
            msg(sender, "&f" + tName + " &7oyuncusunun ev limiti: &f" + getLimit(id));
            return true;
        }

        int n;
        try {
            n = Integer.parseInt(args[1]);
        } catch (NumberFormatException ex) {
            msg(sender, "&cLutfen bir sayi gir.");
            return true;
        }
        if (n < 0 || n > 1000) {
            msg(sender, "&cSayi 0 ile 1000 arasinda olmali.");
            return true;
        }
        data.set("limits." + id, n);
        saveData();
        msg(sender, "&a" + tName + " oyuncusunun ev limiti &f" + n + " &aolarak ayarlandi.");
        return true;
    }

    @Override
    public List<String> onTabComplete(CommandSender sender, Command cmd, String alias, String[] args) {
        String name = cmd.getName().toLowerCase();
        if (args.length != 1) return Collections.emptyList();
        String prefix = args[0].toLowerCase();
        List<String> out = new ArrayList<>();

        if (name.equals("home") || name.equals("delhome")) {
            if (sender instanceof Player p) {
                for (String h : getHomeNames(p.getUniqueId())) {
                    if (h.startsWith(prefix)) out.add(h);
                }
            }
        } else if (name.equals("homelimit") && sender.hasPermission("kyunhomes.admin")) {
            for (Player o : Bukkit.getOnlinePlayers()) {
                if (o.getName().toLowerCase().startsWith(prefix)) out.add(o.getName());
            }
        }
        return out;
    }

    // ------------------------------------------------------------------
    // Menu (GUI)
    // ------------------------------------------------------------------

    private static class HomeMenu implements InventoryHolder {
        final int page;
        Inventory inventory;

        HomeMenu(int page) {
            this.page = page;
        }

        @Override
        public Inventory getInventory() {
            return inventory;
        }
    }

    private ItemStack item(Material m, String name, String... lore) {
        ItemStack it = new ItemStack(m);
        ItemMeta meta = it.getItemMeta();
        meta.displayName(LEGACY.deserialize(name).decoration(TextDecoration.ITALIC, false));
        if (lore.length > 0) {
            List<Component> l = new ArrayList<>();
            for (String s : lore) {
                l.add(LEGACY.deserialize(s).decoration(TextDecoration.ITALIC, false));
            }
            meta.lore(l);
        }
        it.setItemMeta(meta);
        return it;
    }

    private void reopen(Player p, int page) {
        Bukkit.getScheduler().runTask(this, () -> openMenu(p, page));
    }

    private void openMenu(Player p, int page) {
        UUID id = p.getUniqueId();
        List<String> names = getHomeNames(id);
        int limit = getLimit(id);
        int total = Math.max(limit, names.size());
        int pages = Math.max(1, (int) Math.ceil(total / (double) HOME_SLOTS));
        int pg = Math.max(0, Math.min(page, pages - 1));

        HomeMenu holder = new HomeMenu(pg);
        Inventory inv = Bukkit.createInventory(holder, 54,
                LEGACY.deserialize("&8Evlerin (" + (pg + 1) + "/" + pages + ")"));
        holder.inventory = inv;

        for (int i = 0; i < HOME_SLOTS; i++) {
            int idx = pg * HOME_SLOTS + i;
            if (idx >= total) break;
            if (idx < names.size()) {
                String hn = names.get(idx);
                Location loc = data.getLocation(homesPath(id) + "." + hn);
                String where = (loc != null && loc.getWorld() != null)
                        ? loc.getWorld().getName() + " " + loc.getBlockX() + ", " + loc.getBlockY() + ", " + loc.getBlockZ()
                        : "bilinmiyor";
                inv.setItem(i, item(Material.RED_BED, "&a" + hn,
                        "&7" + where, "",
                        "&eSol tik: &fisinlan",
                        "&cShift + sag tik: &fsil"));
            } else {
                inv.setItem(i, item(Material.LIGHT_GRAY_BED, "&7Bos slot",
                        "&eTikla: &fburaya ev kur"));
            }
        }

        ItemStack filler = item(Material.BLACK_STAINED_GLASS_PANE, " ");
        for (int i = HOME_SLOTS; i < 54; i++) inv.setItem(i, filler);
        if (pg > 0) inv.setItem(45, item(Material.ARROW, "&eOnceki sayfa"));
        if (pg < pages - 1) inv.setItem(53, item(Material.ARROW, "&eSonraki sayfa"));
        inv.setItem(49, item(Material.PAPER, "&bEv sayisi: &f" + names.size() + "/" + limit));

        p.openInventory(inv);
    }

    @EventHandler
    public void onClick(InventoryClickEvent e) {
        if (!(e.getInventory().getHolder() instanceof HomeMenu menu)) return;
        e.setCancelled(true);
        if (!(e.getWhoClicked() instanceof Player p)) return;
        if (e.getClickedInventory() == null || !e.getClickedInventory().equals(e.getInventory())) return;

        int slot = e.getSlot();
        int page = menu.page;
        UUID id = p.getUniqueId();
        ItemStack clicked = e.getCurrentItem();

        // sayfa dugmeleri
        if (slot == 45 && clicked != null && clicked.getType() == Material.ARROW) {
            reopen(p, page - 1);
            return;
        }
        if (slot == 53 && clicked != null && clicked.getType() == Material.ARROW) {
            reopen(p, page + 1);
            return;
        }
        if (slot < 0 || slot >= HOME_SLOTS) return;

        int idx = page * HOME_SLOTS + slot;
        List<String> names = getHomeNames(id);

        if (idx < names.size()) {
            String hn = names.get(idx);
            if (e.getClick() == ClickType.SHIFT_RIGHT) {
                deleteHome(id, hn);
                msg(p, "&a'" + hn + "' evi silindi.");
                reopen(p, page);
            } else if (e.getClick() == ClickType.LEFT || e.getClick() == ClickType.SHIFT_LEFT) {
                Location loc = data.getLocation(homesPath(id) + "." + hn);
                if (loc == null || loc.getWorld() == null) {
                    msg(p, "&cBu evin konumu bulunamadi.");
                    return;
                }
                Bukkit.getScheduler().runTask(this, () -> {
                    p.closeInventory();
                    startTeleport(p, loc, hn);
                });
            }
        } else if (idx < getLimit(id)) {
            // bos slota tiklandi: otomatik isimle ev kur
            setHome(p, freeAutoName(id));
            reopen(p, page);
        }
    }
}
