/**
     * 区分tab的礼物列表
     *
     * @param $scene        场景
     * @param $host_uuid
     * @param $account_uuid
     * @param mixed $new_level_version
     * @param mixed $show_heartbeat_gift 私信才用到这个字段：是否展示亲密等级限制礼物
     *
     * @throws ClassNotFoundException
     *
     * @return array
     */
    public function getTabGiftList($scene, $host_uuid, $account_uuid, $new_level_version = 0, $show_heartbeat_gift = 0)
    {
        if (empty($account_uuid)) {
            return $this->setError('ticket_expire');
        }

        $return_gift_list = [];
        $redis            = r();
        $redis->hIncrBy(redis_key('gift_list_stat', date('Ymd')), $scene, 1);
        $redis->expire(redis_key('gift_list_stat', date('Ymd')), 86400 * 7);

        $tab_gift_list  = [];
        $tab_gift_sort  = [];
        $hide_gift_list = [];
        $gift_list      = [];
        //  @var $common_service CommonService
        $common_service = s('Common');
        $now            = $common_service->getRequestTime();

        $platform                 = b('platform_id');
        $app_version              = b('app_version');
        $channel                  = b('channel');
        $oppo_disable_gift_config = c('oppo_disable_gift') ?? [];
        $special_position_info    = [];

        // 许愿礼物id集合
        $wish_gifts_id = [];
        // 套装礼物id集合
        $suit_gifts_id = [];

        switch ($scene) {
            case self::LIVE:
                $gift_list             = $this->getSceneListGiftInfoForLive();
                $service               = SoaClient::getSoa('live_api', 'CommonSoa');
                $live_api_info         = $service->getInfoForGift($account_uuid);
                $gift_discount_info    = $live_api_info['gift_discount_info'];
                $special_position_info = empty($live_api_info['gift_special_position_info']) ? [] : $live_api_info['gift_special_position_info'];
                break;
            case self::CHAT:
                $gift_list = $this->getSceneListGiftInfoForChat();
                // 过滤子场景
                $live_soa            = SoaClient::getSoa('live_api', 'ChatCombinedInfo');
                $chat_info           = $live_soa->batchGetItems($host_uuid, ['room_type', 'sub_type', 'station_key']);
                $scene_child_service = s('AdminGiftCenterSC');
                $scene_child         = $scene_child_service->getChildByType($scene, $chat_info['room_type']);
                if ($scene_child) {
                    $this->filterSceneChild($gift_list, $scene_child);
                }
                if (17 == $chat_info['room_type']) {
                    $sub_type = $chat_info['sub_type'];
                    if (1 == $sub_type) {
                        $station_key = 'shi_fa';
                    } elseif (2 == $sub_type) {
                        $station_key = $chat_info['station_key'];
                    }
                    if (!empty($station_key)) {
                        $this->filterTravelStationGift($station_key, $gift_list);
                    }
                }

                if (19 == $chat_info['room_type']) {
                    $this->filterPartyRoomGift($chat_info['sub_type'], $gift_list);
                }

                $wish_gifts_id = s('Wish')->getAllWishGift(self::CHAT);
                // 过滤
                break;
            case self::CALL:
                $gift_list = $this->getSceneListGiftInfoForCall();
                break;
            case self::POST:
                $gift_list = $this->getSceneListGiftInfoForPost();
                break;
            case self::MSG:
                // 私信请求量太大了,单独开缓存
                $gift_list = $this->getTabGiftCacheWithScene(self::MSG);
                break;
            case self::MSG_FAMILY:
                $gift_list = $this->getSceneListGiftInfoForMsgFamily();
                /*
                // 国庆动效礼物过滤
                $family_activity_soa  = SoaClient::getSoa('family', 'NationalDay2022');
                $national_gift_filter = $family_activity_soa->getNationalDayFilterGift($account_uuid);
                if ($family_activity_soa->hasError()) {
                    $national_gift_filter = [];
                    LOG::e('获取家族国庆动效礼物过滤发生异常! code:' . $family_activity_soa->getErrorCode() . '，msg:' . $family_activity_soa->getErrorMsg());

                    $family_activity_soa->flushError();
                }*/
                $wish_gifts_id = s('Wish')->getAllWishGift(self::MSG_FAMILY);

                $suit_gifts_id = s('GiftSuit')->getAllSuitGids();
                break;
            case self::PEIPEI_GI:
                $gift_list = $this->getSceneListGiftInfoForPeipeiGI();
                break;
            case self::PP_FAMILY:
                $gift_list = $this->getSceneListGiftInfoForPpFamily();
                break;
            case self::CP_CHAT:
                $gift_list = $this->getSceneListGiftInfoForCpChat();
                break;
            default:
                break;
        }

        if (empty($gift_list)) {
            return [
                'gift_list'             => [],
                'hide_gift_list'        => [],
                'special_position_info' => [],
            ];
        }

        $this->sortGiftByCloned($gift_list, $scene, b('cloned'));

        $gift_banner_list = $this->getValidGiftBannerInfo('', $scene);

        $guard_group_service = SoaClient::getSoa('live_api', 'GuardGroup');

        $service = SoaClient::getSoa('live_api', 'LiveConfig');

        $gift_config      = $service->getConfigCache('gift_id_list');
        $un_batch_gift_id = $gift_config['un_batch_gift_id'];

        $gid_list     = array_values(array_unique(array_column($gift_list, 'gid')));
        $diy_gift_tag = Util::getRedisHashCache(redis_key('diy_gift_tag'));

        // 获取礼物对应经验值

        $growth_service = SoaClient::getSoa('live_api', 'GrowthSystem');

        $exp_list = $growth_service->batchGetGiftExpRelation($scene, array_column($gift_list, 'price', 'gid'));

        // 升级礼物
        $filter_air_ship          = [];
        $filter_national_day      = [];
        $account_chat_glory_level = [];
        if (self::CHAT == $scene) {
            // 获取用户需要过滤的魅力飞船礼物(非当前等级的礼物)
            $filter_air_ship = $this->filterAirshipGift($account_uuid);
            // 国庆节解锁礼物
            $filter_national_day = $this->filterChatRoomNationalDayGift($account_uuid);
            // 获取用户的荣耀等级
            $account_chat_glory_level = $this->getChatGloryLevelInfo($account_uuid);
        }

        if (self::MSG_FAMILY == $scene) {
            $filter_air_ship = $this->filterAirshipGift($account_uuid, 3);
        }

        if (!empty($host_uuid)) {
            $is_guard_group = $guard_group_service->batchIsShowGlowStickGift($account_uuid, $host_uuid, $gid_list);
        }

        foreach ($gift_list as $gift) {
            if (1 == $gift['combo_num_onoff'] && !empty($gift['combo_num_info'])) {
                $combo_num_info = json_decode($gift['combo_num_info'], true);
            } else {
                $combo_num_info = null;
            }
            $temp = [
                'gid'                => $gift['gid'],
                'title'              => $gift['title'],
                'introduction'       => $gift['introduction'],
                'price'              => $gift['price'],
                'unlock_level'       => $gift['unlock_level'],
                'gift_url'           => $gift['gift_url'] ?: '',
                'tag_icon'           => $gift['tag_icon'] ?: '',
                'faceu_id'           => $gift['faceu_id'] ?: '',
                'bb_effect_id'       => $gift['bb_effect_id'] ?: '',
                'diy_gift_tag'       => $diy_gift_tag[$gift['gid']] ?: '',
                'unlock_noble'       => $gift['unlock_noble'] ?: '',
                'send_uuid'          => $gift['send_uuid'] ?: '',
                'is_receive_sole'    => (!empty($host_uuid) && false !== strpos($gift['receive_uuid'], $host_uuid)) ? '1' : '0',
                'is_batch'           => is_array($un_batch_gift_id) && in_array($gift['gid'], $un_batch_gift_id) ? '0' : '1',
                'combo_group'        => [],
                'exp'                => (string) $exp_list[$gift['gid']]['exp'] ?: '0',
                'unlock_guard_group' => $gift['unlock_guard_group'] ?: '',
                'discount_ratio'     => 0,
                'combo_num_info'     => $combo_num_info,
                'disabled'           => 0,
                'glory_key'          => '',
                'unlock_fans_group'  => $gift['unlock_fans_group'] ?: '',
                'is_huge_fans'       => $gift['is_huge_fans'] ?: '',
            ];

            if (self::CHAT == $scene && 1 == $gift['is_limit_glory']) {
                if (!in_array($gift['glory_level'], $account_chat_glory_level)) {
                    $temp['disabled'] = 1;
                }
                $temp['glory_key'] = $gift['glory_level'] ?: '';
            }
            // 只有私信 && 用户在心动值体系AB实验为实验组 则才下发等级
            if (self::MSG == $scene && 1 == $show_heartbeat_gift) {
                \LOG::d('私信用户获取礼物tab列表[account_uuid]' . $account_uuid . '[gid]' . $gift['gid'] . '[gift_info]' . json_encode($gift, 256));
                $temp['heartbeat_unlock_level'] = $gift['heartbeat_unlock_level'] ?: 0;
            }

            if (!empty($gift_discount_info) && !empty($discount_ratio = $gift_discount_info[$gift['gid']])) {
                $temp['discount_ratio'] = $discount_ratio;
                $temp['price']          = round($gift['price'] * $discount_ratio / 10);
            }

            // 特殊处理一下两个月光宝盒
            if ('super_000008' == $gift['gid']) {
                $soa_moon_box           = SoaClient::getSoa('live_trade', 'MoonBoxGiftPro');
                $is_first               = $soa_moon_box->isDayFirstLottery($account_uuid);
                $temp['extra_tag_icon'] = $is_first ? c('moon_box_gift_tag') : '';
            }

            if ('super_000011' == $gift['gid']) {
                $soa_star_box           = SoaClient::getSoa('live_trade', 'StarBox');
                $is_first               = $soa_star_box->isDayFirstLottery($account_uuid);
                $temp['extra_tag_icon'] = $is_first ? c('moon_box_gift_tag') : '';
            }

            $this->addGiftBanner($temp, $gift_banner_list, $gift['gid'], $host_uuid);

            // 旧版本不现实财富等级专属礼物
            if (empty($new_level_version) && !empty($gift['unlock_level'])) {
                $hide_gift_list[] = $temp;
                continue;
            }

            if (empty($gift['tab_id'])) {
                $hide_gift_list[] = $temp;
                continue;
            }

            if (!empty($host_uuid) && !$is_guard_group[$gift['gid']]) {
                $hide_gift_list[] = $temp;
                continue;
            }

            if (('oppo' == $channel) && in_array($gift['gid'], $oppo_disable_gift_config)) {
                $hide_gift_list[] = $temp;
                continue;
            }

            if (!empty($gift['receive_uuid']) && !empty($host_uuid) && false === strpos($gift['receive_uuid'], $host_uuid)) {
                $hide_gift_list[] = $temp;
                continue;
            }

            if (!empty($gift['send_uuid']) && !empty($account_uuid) && false === strpos($gift['send_uuid'], $account_uuid)) {
                $hide_gift_list[] = $temp;
                continue;
            }

            if ($app_version < $gift['android_version'] && '1' == $platform) {
                $hide_gift_list[] = $temp;
                continue;
            }

            // 苹果
            if ($app_version < $gift['ios_version'] && ('2' == $platform || '3' == $platform)) {
                $hide_gift_list[] = $temp;
                continue;
            }

            // 隐藏魅力飞船-非当前等级礼物
            if (!empty($filter_air_ship) && in_array($gift['gid'], $filter_air_ship)) {
                $hide_gift_list[] = $temp;
                continue;
            }

            // 隐藏国庆未解锁礼物
            if (!empty($filter_national_day) && in_array($gift['gid'], $filter_national_day)) {
                $hide_gift_list[] = $temp;
                continue;
            }

            // 粉丝团和守护团相互隐藏
            if ($new_level_version >= 2 && !empty($temp['unlock_guard_group'])) {
                $hide_gift_list[] = $temp;
                continue;
            }

            // 粉丝团和守护团相互隐藏
            if ($new_level_version < 2 && !empty($temp['unlock_fans_group'])) {
                $hide_gift_list[] = $temp;
                continue;
            }

            if ($new_level_version < 3 && !empty($temp['is_huge_fans'])) {
                $hide_gift_list[] = $temp;
                continue;
            }

            /*
            // 国庆活动结束后代码可删除
            if (!empty($national_gift_filter) && in_array($gift['gid'], $national_gift_filter)) {
                $hide_gift_list[] = $temp;
                continue;
            }*/

            // 聊天室不下发专属主播，改为送礼接口判断是否在专属主播房间
            if (self::CHAT == $scene) {
                $temp['is_receive_sole'] = 0;
            } elseif (self::MSG == $scene || self::MSG_FAMILY == $scene) {
                // 社区宝盒特殊属性
                $box_gift_map = $this->getForumBoxGiftMap();
                if (in_array($temp['gid'], $box_gift_map)) {
                    // 月光宝盒
                    $temp['is_moon_box'] = 1;
                    $temp['moon_box']    = array_flip($box_gift_map)[$temp['gid']];
                }
            }
            // 彩蛋
            if (!empty($gift['egg_status'])) {
                $gift_comb_info = $this->getSingleInfo($scene, $gift['gid'], ['combo_type', 'combo_info', 'combo_info_multi']);
                if (empty($gift_comb_info['combo_type']) && !empty($gift_comb_info['combo_info'])) {
                    // 逐个设置
                    $temp['combo_group'] = array_merge($temp['combo_group'], array_keys(json_decode($gift_comb_info['combo_info'], true)));
                } elseif (!empty($gift_comb_info['combo_info_multi'])) {
                    // 批量设置
                    $temp['combo_group'] = array_merge($temp['combo_group'], array_keys(json_decode($gift_comb_info['combo_info_multi'], true)));
                }
            }
            // 连击奖励, 开启且时间满足
            if (!empty($gift['award_status'])) {
                $gift_award_info = $this->getSingleInfo($scene, $gift['gid'], ['award_start_time', 'award_end_time', 'award_type', 'award_info']);
                if ($now >= $gift_award_info['award_start_time'] && $now <= $gift_award_info['award_end_time']) {
                    if (empty($gift_award_info['award_type']) && !empty($gift_award_info['award_info'])) {
                        // 逐个设置
                        $temp['combo_group'] = array_merge($temp['combo_group'], array_keys(json_decode($gift_award_info['award_info'], true)));
                    } elseif (!empty($gift_award_info['award_info_multi'])) {
                        // 批量设置
                        $temp['combo_group'] = array_merge($temp['combo_group'], array_keys(json_decode($gift_award_info['award_info_multi'], true)));
                    }
                }
            }
            !empty($temp['combo_group']) && $temp['combo_group'] = array_values(array_unique($temp['combo_group']));

            // 是否许愿礼物
            $temp['is_wish'] = !empty($wish_gifts_id) && in_array($temp['gid'], $wish_gifts_id) ? 1 : 0;

            // 是否套装礼物
            if (self::MSG_FAMILY == $scene) {
                $temp['is_gift_suit'] = !empty($suit_gifts_id) && in_array($temp['gid'], $suit_gifts_id) ? 1 : 0;
            }

            $tab_gift_list[$gift['tab_id']][] = $temp;
            $tab_gift_sort[$gift['tab_id']][] = $gift['sort'];
        }

        $tab_list = $this->getGiftTabCache();

        // 获取用户是否极简版本
        $service       = SoaClient::getSoa('live_api', 'Room');
        $hidden_tab_id = c('hidden_tab_id') ?? [];

        /** @var AccountService $account_service */
        $account_service = s('Account');
        $host_info       = $account_service->getAccountInfoByUuId($host_uuid, ['sex_type']);

        $show_version = $service->getAccountPanelType($host_uuid, $account_uuid);
        if (!empty($tab_list)) {
            $tab_ids         = array_column($tab_list, 'id');
            $pass_group_id   = $this->handleGroup($tab_list, $account_uuid);
            $tab_banner_info = s('Gift')->getValidTabBannerInfo($tab_ids, $scene);
            foreach ($tab_list as $tab) {
                $temp_list = $tab_gift_list[$tab['id']];
                if (empty($temp_list)) {
                    continue;
                }
                // 私信 && 非心动值体系AB实验组 && 配置的隐藏tab 则设置隐藏掉配置的tab【简单来说就是旧版本不展示】
                if (self::MSG === (int) $scene && empty($show_heartbeat_gift) && in_array($tab['id'], $hidden_tab_id)) {
                    continue;
                }
                // 私信 && 心动值体系AB实验组 && 配置的隐藏tab 新版同性不展示配置的tab
                if (self::MSG === (int) $scene && 1 == $show_heartbeat_gift && in_array($tab['id'], $hidden_tab_id) && $host_info['sex_type'] == b('gender')) {
                    continue;
                }

                array_multisort($tab_gift_sort[$tab['id']], SORT_DESC, $temp_list);
                $is_hidden = self::LIVE === (int) $scene && in_array($show_version, [3, 4]) && !empty($tab['is_hidden']) ? '1' : '0';
                if ($new_level_version >= 2 && '守护团' == $tab['title']) {
                    $tab['title'] = '粉丝团';
                }

                if (self::LIVE === (int) $scene && !empty($tab['display_start_time']) && !empty($tab['display_end_time'])) {
                    if (time() < $tab['display_start_time'] || $tab['display_end_time'] < time()) {
                        continue;
                    }
                }

                if (self::LIVE === (int) $scene && -1 != $tab['group_id'] && !empty($tab['group_id'])) {
                    // 过滤人群
                    if (empty($pass_group_id)) {
                        continue;
                    }

                    if (!in_array($tab['group_id'], $pass_group_id)) {
                        continue;
                    }
                }

                $return_gift_list['gift_list'][] = [
                    'tab_name' => $tab['title'],
                    'tab_id'   => $tab['id'],
                    // 非极简版本且列表隐藏时处理tab隐藏功能，目前直播特有
                    'is_hidden'            => $is_hidden,
                    'list'                 => $temp_list,
                    'tab_banner_icon'      => empty($tab_banner_info[$tab['id']]['banner_icon']) ? '' : $tab_banner_info[$tab['id']]['banner_icon'],
                    'tab_banner_lottie_id' => empty($tab_banner_info[$tab['id']]['banner_lottie_id']) ? '' : $tab_banner_info[$tab['id']]['banner_lottie_id'],
                    'tab_banner_schema'    => empty($tab_banner_info[$tab['id']]['banner_schema']) ? '' : $tab_banner_info[$tab['id']]['banner_schema'],
                ];
            }
        }

        $return_gift_list['hide_gift_list'] = $hide_gift_list;
        $return_gift_list['free_gift_info'] = array_flip(c('free_gift_id'));
        if (empty($return_gift_list)) {
            return null;
        }

        if (self::POST == $scene) {
            $msg_array                    = s('ForumGift')->getForumGiftMessage();
            $index                        = $msg_array ? mt_rand(0, count($msg_array) - 1) : 0;
            $return_gift_list['gift_msg'] = strval($msg_array[$index]['gift_message'] ?? '');
        }

        // 特殊展示位
        $return_gift_list['special_position_info'] = $special_position_info;
        // 许愿礼物特效
        $return_gift_list['wish_gift_effect_id'] = c('wish_gift.effect_id');

        return $return_gift_list;
    }
