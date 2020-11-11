### 1. 按照上次 AIO-3399J 的配置之后，屏幕不亮。
* 经过查找原因，发现背光启动之后一段时间后被关闭。
* 原因如下：
* 触发了 /drivers/gpu/drm/panel/panel-simple.c 里面的
* panel_simple_disable 函数。
```
 536 static int panel_simple_disable(struct drm_panel *panel)
 537 {
 538     struct panel_simple *p = to_panel_simple(panel);
 539
 540     printk("file: %s , func: %s, line: %d ------------------debug\n", __FIL     E__, __func__, __LINE__);
 541
 542     if (!p->enabled)
 543         return 0;
 544
 545     backlight_disable(p->backlight);
        // 这个地方关闭了 背光，所以配置了背光 enable 也是没有用的
 546
 547     if (p->desc && p->desc->delay.disable)
 548         panel_simple_sleep(p->desc->delay.disable);
 549
 550     p->enabled = false;
 551
 552     return 0;
 553 }
 
 695 static const struct drm_panel_funcs panel_simple_funcs = {
 696     .loader_protect = panel_simple_loader_protect,
 697     .disable = panel_simple_disable,
 698     .unprepare = panel_simple_unprepare,
 699     .prepare = panel_simple_prepare,
 700     .enable = panel_simple_enable,
 701     .get_modes = panel_simple_get_modes,
 702     .get_timings = panel_simple_get_timings,
 703 };

 874     drm_panel_init(&panel->base);
 875     panel->base.dev = dev;
 876     panel->base.funcs = &panel_simple_funcs;
```

### 2. Kernel LOG 分析
```
[    4.015664] dw-mipi-dsi ff960000.dsi: final DSI-Link bandwidth: 752 x 4 Mbps
[    4.018252] file: drivers/gpu/drm/panel/panel-simple.c , func: panel_simple_disable, line: 540 ------------------debug
[    4.018303] pwm_backlight_update_status ----
[    4.018334] gpio set name : enable  (null) ------------ value : 0
```
* 最后这行 drivers/gpu/drm/rockchip/dw-mipi-dsi.c 里面的 dw_mipi_dsi_encoder_enable 函数。
```
1359 static void dw_mipi_dsi_encoder_enable(struct drm_encoder *encoder)
1360 {
1361     struct dw_mipi_dsi *dsi = encoder_to_dsi(encoder);
1362     unsigned long lane_rate = dw_mipi_dsi_get_lane_rate(dsi);
1363
1364     if (dsi->dphy.phy)
1365         dw_mipi_dsi_set_hs_clk(dsi, lane_rate);
1366     else
1367         dw_mipi_dsi_set_pll(dsi, lane_rate);
1368
1369     dev_info(dsi->dev, "final DSI-Link bandwidth: %u x %d Mbps\n",
1370          dsi->lane_mbps, dsi->slave ? dsi->lanes * 2 : dsi->lanes);
         // 这个是上面的日志打印的地方
1371
1372     dw_mipi_dsi_vop_routing(dsi);
1373     dw_mipi_dsi_pre_enable(dsi);
1374
1375     if (dsi->panel)
1376         drm_panel_prepare(dsi->panel);
1377
1378     dw_mipi_dsi_enable(dsi);
1379
1380     if (dsi->panel)
1381         drm_panel_enable(dsi->panel);
1382 }
```

* 猜测可能调用 disable 的地方
```
1451 static const struct drm_encoder_helper_funcs
1452 dw_mipi_dsi_encoder_helper_funcs = {
1453     .loader_protect = dw_mipi_dsi_encoder_loader_protect,
1454     .mode_set = dw_mipi_dsi_encoder_mode_set,
1455     .enable = dw_mipi_dsi_encoder_enable,
1456     .disable = dw_mipi_dsi_encoder_disable,
1457     .atomic_check = dw_mipi_dsi_encoder_atomic_check,
1458 };


```
* dw_mipi_dsi_encoder_disable
```
1265 static void dw_mipi_dsi_encoder_disable(struct drm_encoder *encoder)
1266 {
1267     struct dw_mipi_dsi *dsi = encoder_to_dsi(encoder);
1268
1269     if (dsi->panel)
1270         drm_panel_disable(dsi->panel);  
            // 这里调用 panel_simple_disable 函数
            // /drivers/gpu/drm/panel/panel-simple.c
1271
1272     dw_mipi_dsi_disable(dsi);
1273
1274     if (dsi->panel)
1275         drm_panel_unprepare(dsi->panel);
1276
1277     dw_mipi_dsi_post_disable(dsi);
1278 }
```
* include/drm/drm_panel.h
* drm_panel_disable
```
 98 static inline int drm_panel_disable(struct drm_panel *panel)
 99 {
100     if (panel && panel->funcs && panel->funcs->disable)
101         return panel->funcs->disable(panel);
102
103     return panel ? -ENOSYS : -EINVAL;
104 }
```

* dw_mipi_dsi_encoder_helper_funcs 这个函数结构体又是在哪被应用呢：
```
1534 static int dw_mipi_dsi_register(struct drm_device *drm,
1535                       struct dw_mipi_dsi *dsi)
1536 {
1537     struct drm_encoder *encoder = &dsi->encoder;
1538     struct drm_connector *connector = &dsi->connector;
1539     struct device *dev = dsi->dev;
1540     int ret;
1541
1542     encoder->possible_crtcs = drm_of_find_possible_crtcs(drm,
1543                                  dev->of_node);
1544     /*
1545      * If we failed to find the CRTC(s) which this encoder is
1546      * supposed to be connected to, it's because the CRTC has
1547      * not been registered yet.  Defer probing, and hope that
1548      * the required CRTC is added later.
1549      */
1550     if (encoder->possible_crtcs == 0)
1551         return -EPROBE_DEFER;
1552
1553     drm_encoder_helper_add(&dsi->encoder,
1554                    &dw_mipi_dsi_encoder_helper_funcs);
                        // 上面的 dw_mipi_dsi_encoder_helper_funcs
                        // 又有下面的赋值
                        // 那么调用应该就是
            // &dsi->encoder->helper_private->disable
            // 就是执行了 dw_mipi_dsi_encoder_disable 这个函数
1555     ret = drm_encoder_init(drm, &dsi->encoder, &dw_mipi_dsi_encoder_funcs,
1556              DRM_MODE_ENCODER_DSI, NULL);
    /// ...
    }
```
* include/drm/drm_crtc_helper.h
* drm_encoder_helper_add
```
219 static inline void drm_encoder_helper_add(struct drm_encoder *encoder,
220                       const struct drm_encoder_helper_funcs *funcs)
221 {
222     encoder->helper_private = funcs;
223 }
```

*  这个 dw_mipi_dsi_register 函数的调用也还在这个文件夹里面
*  drivers/gpu/drm/rockchip/dw-mipi-dsi.c
*  dw_mipi_dsi_bind
```
1604 static int dw_mipi_dsi_bind(struct device *dev, struct device *master,
1605                  void *data)
1606 {
1607     struct drm_device *drm = data;
1608     struct dw_mipi_dsi *dsi = dev_get_drvdata(dev);
1609     int ret;
1610
1611     ret = dw_mipi_dsi_dual_channel_probe(dsi);
1612     if (ret)
1613         return ret;
1614
1615     if (dsi->master)
1616         return 0;
1617
1618     dsi->panel = of_drm_find_panel(dsi->client);
1619     if (!dsi->panel) {
1620         dsi->bridge = of_drm_find_bridge(dsi->client);
1621         if (!dsi->bridge)
1622             return -EPROBE_DEFER;
1623     }
1624    // 这里就是执行到了上面的
        // static int dw_mipi_dsi_register(struct drm_device *drm, 
        //    struct dw_mipi_dsi *dsi)
1625     ret = dw_mipi_dsi_register(drm, dsi);
1626     if (ret) {
1627         dev_err(dev, "Failed to register mipi_dsi: %d\n", ret);
1628         return ret;
1629     }
1630
1631     dev_set_drvdata(dev, dsi);
1632
1633     pm_runtime_enable(dev);
1634     if (dsi->slave)
1635         pm_runtime_enable(dsi->slave->dev);
1636
1637     return ret;
1638 }
```
* dw_mipi_dsi_ops
```
1653 static const struct component_ops dw_mipi_dsi_ops = {
1654     .bind   = dw_mipi_dsi_bind,
1655     .unbind = dw_mipi_dsi_unbind,
1656 };
```
* 这个 dw_mipi_dsi_ops 函数结构体就直接被 probe 调用

```
1814 static int dw_mipi_dsi_probe(struct platform_device *pdev)
1815 {
        // 。。。 。。。
        // 这里直接被调用
1908     ret = component_add(dev, &dw_mipi_dsi_ops);
1909     if (ret)
1910         mipi_dsi_host_unregister(&dsi->dsi_host);
1911
1912     printk("file: %s , func: %s, line: %d, ret : %d ------------------debug     \n", __FILE__, __func__, __LINE__, ret);
1913
1914
1915     return ret;
1916 }
```

### 3. dw_mipi_dsi_encoder_disable 调用的地方
* static void dw_mipi_dsi_encoder_disable(struct drm_encoder *encoder)
* 看参数类型 struct drm_encoder, 在 driver/drm 文件夹里面找。
* 或者在 include/drm 里面搜
* drivers/gpu/drm/drm_atomic_helper.c
```
 639         if (connector->state->crtc && funcs->prepare)
 640             funcs->prepare(encoder);
 641         else if (funcs->disable)
 642         {
 643             printk("file: %s , func: %s, line: %d ------------------debug\n     ", __FILE__, __func__, __LINE__);
 644             funcs->disable(encoder);
 645         }
 646         else
 647             funcs->dpms(encoder, DRM_MODE_DPMS_OFF);
```
* 这里又是为什么被调用呢？？？
* drivers/gpu/drm/drm_atomic_helper.c
```
 829 void drm_atomic_helper_commit_modeset_disables(struct drm_device *dev,
 830                            struct drm_atomic_state *old_state)
 831 {
 832     disable_outputs(dev, old_state);
 833
 834     drm_atomic_helper_update_legacy_modeset_state(dev, old_state);
 835
 836     crtc_set_mode(dev, old_state);
 837 }
 838 EXPORT_SYMBOL(drm_atomic_helper_commit_modeset_disables);
```

* drivers/gpu/drm/rockchip/rockchip_drm_fb.c
* rockchip_atomic_commit_complete
```
287     printk("file: %s , func: %s, line: %d ------------------debug\n", __FILE    __, __func__, __LINE__);
288     drm_atomic_helper_commit_modeset_disables(dev, state);
```

* 
```
328 int rockchip_drm_atomic_commit(struct drm_device *dev,
329                    struct drm_atomic_state *state,
330                    bool async)
331 {
    //... ...
    /// 这里 async 如果为 false ， 直接执行这个操作 disable 操作
367     if (async) {
368         mutex_lock(&private->commit_lock);
369
370         flush_work(&private->commit_work);
371         WARN_ON(private->commit);
372         private->commit = commit;
373         schedule_work(&private->commit_work);
374
375         mutex_unlock(&private->commit_lock);
376     } else {
377         printk("file: %s , func: %s, line: %d ------------------debug\n", __    FILE__, __func__, __LINE__);
378         rockchip_atomic_commit_complete(commit);
379     }
380
381     return 0;
382 }
```
* drivers/gpu/drm/rockchip/rockchip_drm_fb.c
* rockchip_drm_mode_config_funcs
```
384 static const struct drm_mode_config_funcs rockchip_drm_mode_config_funcs = {
385     .fb_create = rockchip_user_fb_create,
386     .output_poll_changed = rockchip_drm_output_poll_changed,
387     .atomic_check = drm_atomic_helper_check,
388     .atomic_commit = rockchip_drm_atomic_commit,
389 };
```
```
405 void rockchip_drm_mode_config_init(struct drm_device *dev)
406 {
407     dev->mode_config.min_width = 0;
408     dev->mode_config.min_height = 0;
409
410     /*
411      * set max width and height as default value(4096x4096).
412      * this value would be used to check framebuffer size limitation
413      * at drm_mode_addfb().
414      */
415     dev->mode_config.max_width = 8192;
416     dev->mode_config.max_height = 8192;
417
418     dev->mode_config.funcs = &rockchip_drm_mode_config_funcs;
419 }
```

* 感觉又回来了 drivers/gpu/drm/drm_atomic.c
* drm_atomic_commit
```
1458 int drm_atomic_commit(struct drm_atomic_state *state)
1459 {
1460     struct drm_mode_config *config = &state->dev->mode_config;
1461     int ret;
1462
1463     ret = drm_atomic_check_only(state);
1464     if (ret)
1465         return ret;
1466
1467     DRM_DEBUG_ATOMIC("commiting %p\n", state);
1468
1469     printk("file: %s , func: %s, line: %d ------------------debug\n", __FIL     E__, __func__, __LINE__);
1470
1471     return config->funcs->atomic_commit(state->dev, state, false);
1472 }
1473 EXPORT_SYMBOL(drm_atomic_commit);
```

* 再回来 drivers/gpu/drm/rockchip/rockchip_drm_drv.c
```
 810     printk("file: %s , func: %s, line: %d ------------------debug\n", __FIL     E__, __func__, __LINE__);
 811     ret = drm_atomic_commit(state);
```