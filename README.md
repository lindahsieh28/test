# test

## Claude Code 外掛設定

本專案的 `.claude/settings.json` 已宣告 `anthropics/knowledge-work-plugins` 外掛市集，並預設啟用以下 17 個外掛，之後開啟的 Claude Code 工作階段會自動載入：

productivity、enterprise-search、cowork-plugin-management、sales、finance、data、legal、marketing、customer-support、product-management、bio-research、engineering、human-resources、design、operations、small-business、pdf-viewer

首次使用時，Claude Code 會提示是否信任此專案並安裝市集，選擇同意即可。若要停用某個外掛，將 `enabledPlugins` 中對應項目改為 `false` 或刪除即可。
