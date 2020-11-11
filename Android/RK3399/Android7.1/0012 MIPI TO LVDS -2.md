### 1. drivers/gpu/drm/panel/panel-simple.c
* 代码分析
```
2448 static int __init panel_simple_init(void)
2449 {
2450     int err;
2451
2452     err = platform_driver_register(&panel_simple_platform_driver);
2453     if (err < 0)
2454         return err;
2455
2456     if (IS_ENABLED(CONFIG_DRM_MIPI_DSI)) {
2457         err = mipi_dsi_driver_register(&panel_simple_dsi_driver);
2458         if (err < 0)
2459             return err;
2460     }
2461
2462     return 0;
2463 }
2464 module_init(panel_simple_init);
```
```
2356 static int panel_simple_dsi_probe(struct mipi_dsi_device *dsi)
2357 {
2358     struct device *dev = &dsi->dev;
2359     struct panel_simple *panel;
2360     const struct panel_desc_dsi *desc;
2361     const struct of_device_id *id;
2362     const struct panel_desc *pdesc;
2363     int err;
2364     u32 val;
2365
2366     id = of_match_node(dsi_of_match, dev->of_node);
2367     if (!id)
2368         return -ENODEV;
2369
2370     desc = id->data;
2371
2372     if (desc) {
2373         dsi->mode_flags = desc->flags;
2374         dsi->format = desc->format;
2375         dsi->lanes = desc->lanes;
2376         pdesc = &desc->desc;
2377     } else {
2378         pdesc = NULL;
2379     }
2380
2381     err = panel_simple_probe(dev, pdesc);
```
```
 755 static int panel_simple_probe(struct device *dev, const struct panel_desc *     desc)
 756 {
 757     struct device_node *backlight, *ddc;
 758     struct panel_simple *panel;
 759     struct panel_desc *of_desc;
 760     const char *cmd_type;
 761     u32 val;
 762     int err;
 763
 764     panel = devm_kzalloc(dev, sizeof(*panel), GFP_KERNEL);
 765     if (!panel)
 766         return -ENOMEM;
 767
 768     if (!desc)
 769         of_desc = devm_kzalloc(dev, sizeof(*of_desc), GFP_KERNEL);
 770     else
 771         of_desc = devm_kmemdup(dev, desc, sizeof(*of_desc), GFP_KERNEL);
 773     if (!of_property_read_u32(dev->of_node, "bus-format", &val))
 774         of_desc->bus_format = val;
 775     if (!of_property_read_u32(dev->of_node, "bpc", &val))
 776         of_desc->bpc = val;
 777     if (!of_property_read_u32(dev->of_node, "prepare-delay-ms", &val))
 778         of_desc->delay.prepare = val;
 779     if (!of_property_read_u32(dev->of_node, "enable-delay-ms", &val))
 780         of_desc->delay.enable = val;
 781     if (!of_property_read_u32(dev->of_node, "disable-delay-ms", &val))
 782         of_desc->delay.disable = val;
 783     if (!of_property_read_u32(dev->of_node, "unprepare-delay-ms", &val))
 784         of_desc->delay.unprepare = val;
 785     if (!of_property_read_u32(dev->of_node, "reset-delay-ms", &val))
 786         of_desc->delay.reset = val;
 787     if (!of_property_read_u32(dev->of_node, "init-delay-ms", &val))
 788         of_desc->delay.init = val;
 789     if (!of_property_read_u32(dev->of_node, "width-mm", &val))
 790         of_desc->size.width = val;
 791     if (!of_property_read_u32(dev->of_node, "height-mm", &val))
 792         of_desc->size.height = val;
 793
 794     panel->enabled = false;
 795     panel->prepared = false;
 796     panel->desc = of_desc;
 797     panel->dev = dev;
 798
 799     err = panel_simple_get_cmds(panel);
```
```
 341 static int panel_simple_get_cmds(struct panel_simple *panel)
 342 {
 343     const void *data;
 344     int len;
 345     int err;
 346
 347     data = of_get_property(panel->dev->of_node, "panel-init-sequence",
 348                    &len);
 349
 350     if (data) {
 351         panel->on_cmds = devm_kzalloc(panel->dev,
 352                           sizeof(*panel->on_cmds),
 353                           GFP_KERNEL);
 354         if (!panel->on_cmds)
 355             return -ENOMEM;
 356
 357         err = panel_simple_parse_cmds(panel->dev, data, len,
 358                           panel->on_cmds);
 359         if (err) {
 360             dev_err(panel->dev, "failed to parse panel init sequence\n");
 361             return err;
 362         }
 363     }
 364
 365     data = of_get_property(panel->dev->of_node, "panel-exit-sequence",
 366                    &len);
 367     if (data) {
 368         panel->off_cmds = devm_kzalloc(panel->dev,
 369                            sizeof(*panel->off_cmds),
 370                            GFP_KERNEL);
 371         if (!panel->off_cmds)
 372             return -ENOMEM;
 373
 374         err = panel_simple_parse_cmds(panel->dev, data, len,
 375                           panel->off_cmds);
 376         if (err) {
 377             dev_err(panel->dev, "failed to parse panel exit sequence\n");
 378             return err;
 379         }
 380     }
 381     return 0;
 382 }
```

### 2. 现在找到的点是 没有 执行send mipi cmd
* drivers/gpu/drm/panel/panel-simple.c
```
 701 static const struct drm_panel_funcs panel_simple_funcs = {
 702     .loader_protect = panel_simple_loader_protect,
 703     .disable = panel_simple_disable,
 704     .unprepare = panel_simple_unprepare,
 705     .prepare = panel_simple_prepare,
            // 上面这个没被执行，这个是执行命令的一个过程
            
 706     .enable = panel_simple_enable,
            // enable  也没被执行
            
 707     .get_modes = panel_simple_get_modes,
            // get mode 被执行了。
            
 708     .get_timings = panel_simple_get_timings,
 709 };
```
```
 591 static int panel_simple_prepare(struct drm_panel *panel)
 592 {
 593     struct panel_simple *p = to_panel_simple(panel);
 594     int err;
 595
 596     if (p->prepared)
 597         return 0;
 598
 599     printk("file: %s , func: %s, line: %d ------------------debug\n", __FIL     E__, __func__, __LINE__);
 600     err = panel_simple_regulator_enable(panel);
 601     if (err < 0) {
 602         dev_err(panel->dev, "failed to enable supply: %d\n", err);
 603         return err;
 604     }
        // ... ...
         624     if (p->on_cmds) {
 625         if (p->dsi)
 626             err = panel_simple_dsi_send_cmds(p, p->on_cmds);
 627         else if (p->cmd_type == CMD_TYPE_SPI)
 628             err = panel_simple_spi_send_cmds(p, p->on_cmds);
                // 这里执行命令，但是这里没有执行，
                // 这里的代码是初始化 TC358775 的
                // 没有执行这个，TC358775 所有的参数都没有被初始化
 629         if (err)
 630             dev_err(p->dev, "failed to send on cmds\n");
 631     }
 632
 633     p->prepared = true;
 634
 635     return 0;
 636 }
```
* probe 里面注册这个 funcs 结构体
```
 880     drm_panel_init(&panel->base);
 881     panel->base.dev = dev;
 882     panel->base.funcs = &panel_simple_funcs;
 883
```

### 3. 找 RK drivers/gpu/drm/rockchip/dw-mipi-dsi.c
* dw_mipi_dsi_encoder_enable
```
1363 static void dw_mipi_dsi_encoder_enable(struct drm_encoder *encoder)
1364 {
1365     struct dw_mipi_dsi *dsi = encoder_to_dsi(encoder);
1366     unsigned long lane_rate = dw_mipi_dsi_get_lane_rate(dsi);
1367
1368     printk("file: %s , func: %s, line: %d ------------------debug\n", __FIL     E__, __func__, __LINE__);
1369     if (dsi->dphy.phy)
1370         dw_mipi_dsi_set_hs_clk(dsi, lane_rate);
1371     else
1372         dw_mipi_dsi_set_pll(dsi, lane_rate);
1373
1374     dev_info(dsi->dev, "final DSI-Link bandwidth: %u x %d Mbps\n",
1375          dsi->lane_mbps, dsi->slave ? dsi->lanes * 2 : dsi->lanes);
1376
1377     dw_mipi_dsi_vop_routing(dsi);
1378     dw_mipi_dsi_pre_enable(dsi);
1379
1380     if (dsi->panel)
1381         drm_panel_prepare(dsi->panel);
            // 找到执行  simple panel panel_simple_prepare()函数的地方
1382
1383     dw_mipi_dsi_enable(dsi);
1384
1385     if (dsi->panel)
1386         drm_panel_enable(dsi->panel);
            // simple panel 里面的 panel_simple_enable() 应该也是这个地方
1387 }
```
* include/drm/drm_panel.h
```
114 static inline int drm_panel_prepare(struct drm_panel *panel)
115 {
116     if (panel && panel->funcs && panel->funcs->prepare)
117         return panel->funcs->prepare(panel);
118
119     return panel ? -ENOSYS : -EINVAL;
120 }
121
122 static inline int drm_panel_enable(struct drm_panel *panel)
123 {
124     if (panel && panel->funcs && panel->funcs->enable)
125         return panel->funcs->enable(panel);
126
127     return panel ? -ENOSYS : -EINVAL;
128 }
```

* 回到 RK 的 drivers/gpu/drm/rockchip/dw-mipi-dsi.c
* 找 enable 为什么没被执行。