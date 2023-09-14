# Redis bitmap 实现签到

## 按自然月

```java
import cn.hutool.core.collection.CollUtil;
import com.botpy.vosp.common.util.RedisUtil;
import com.botpy.vosp.common.util.SpringUtils;
import lombok.extern.slf4j.Slf4j;

import java.time.LocalDate;
import java.time.format.DateTimeFormatter;
import java.util.List;
import java.util.Map;
import java.util.TreeMap;

/**
 * 签到工具类
 * 1. offset: 此类中方法 offset 参数
 *      语义{ 偏移量 }
 *      意义{ 表示当月的第n天 }
 *      -1的原因{ redis bitmap 中 offset 从 0 开始，月中天从 1 开始 }
 *
 * 2. setBit: bool 结果取反的原因
 *      https://github.com/AgeFades/AgeFades-Note/blob/master/%E5%BC%82%E5%B8%B8%E6%89%8B%E5%86%8C/Redis%E9%97%AE%E9%A2%98%E9%9B%86%E9%94%A6.md
 *
 * 3. 参考链接
 *      https://www.cnblogs.com/liujiduo/p/10396020.html
 * @author DuChao
 * @date 2020/12/2 2:23 下午
 */
@Slf4j
public class SignUtil {

    private static final RedisUtil redisUtil = SpringUtils.getBean(RedisUtil.class);

    /**
     * 客户今日签到
     *
     * @param customerId 客户id
     * @return 是否签到成功
     */
    public static Boolean doSign(String customerId) {
        return !redisUtil.setBit(buildSignKey(customerId), LocalDate.now().getDayOfMonth() - 1, true);
    }

    /**
     * 检查客户今日是否签到
     *
     * @param customerId 客户id
     * @return 是否签到
     */
    public static Boolean checkSign(String customerId) {
        return redisUtil.getBit(buildSignKey(customerId), LocalDate.now().getDayOfMonth() - 1);
    }

    /**
     * 获取客户当月签到天数
     *
     * @param customerId 客户id
     * @return 客户当月签到天数
     */
    public static Long getSignCount(String customerId) {
        return redisUtil.bitCount(buildSignKey(customerId));
    }

    /**
     * 获取客户当前已连续签到天数
     *
     * @param customerId 客户id
     * @return 客户当前已连续签到天数
     */
    public static int getContinuousSignCount(String customerId) {
        int result = 0;

        int dayOfMonth = LocalDate.now().getDayOfMonth();
        List<Long> list = redisUtil.bitField(buildSignKey(customerId), dayOfMonth, 0);
        if (CollUtil.isNotEmpty(list)) {
            long offsetValue = list.get(0) == null ? 0 : list.get(0);
            for (int i = 0; i < dayOfMonth; i++) {
                if (offsetValue >> 1 << 1 == offsetValue) {
                    if (i > 0) break;
                } else {
                    ++result;
                }
                offsetValue >>= 1;
            }
        }
        return result;
    }

    /**
     * 获取客户当月签到情况月历
     *
     * @param customerId 客户id
     * @return 客户当月签到情况月历
     */
    public static Map<String, Boolean> getSignInfo(String customerId) {
        Map<String, Boolean> signInfo = new TreeMap<>();

        LocalDate today = LocalDate.now();

        List<Long> list = redisUtil.bitField(buildSignKey(customerId), today.lengthOfMonth(), 0);
        if (CollUtil.isNotEmpty(list)) {
            // 由低位到高位，为0表示未签，为1表示已签
            long v = list.get(0) == null ? 0 : list.get(0);
            for (int i = today.lengthOfMonth(); i > 0; i--) {
                LocalDate d = today.withDayOfMonth(i);
                signInfo.put(d.format(DateTimeFormatter.ISO_LOCAL_DATE), v >> 1 << 1 != v);
                v >>= 1;
            }
        }

        return signInfo;
    }

    /**
     * 获取客户当月首次签到的日期
     *
     * @param customerId 客户id
     * @return 客户当月首次签到的日期
     */
    public static LocalDate getFirstSignDate(String customerId) {
        Long bitPos = redisUtil.bitPos(buildSignKey(customerId));
        return bitPos < 0 ? null : LocalDate.now().withDayOfMonth((int) (bitPos + 1));
    }

    /**
     * 构造签到key, 格式举例: { sign:1001:202012 | 表示客户id为1001的客户，在2020年12月份签到数据存储的 key }
     *
     * @param customerId 客户id
     * @return 签到key
     */
    private static String buildSignKey(String customerId) {
        return String.format("sign:%s:%s", customerId, LocalDate.now().format(DateTimeFormatter.ofPattern("yyyyMM")));
    }

}
```

## 按7天连续签到

```java
import cn.hutool.core.collection.CollUtil;
import cn.hutool.core.convert.Convert;
import cn.hutool.core.date.LocalDateTimeUtil;
import com.botpy.vosp.common.util.RedisUtil;
import com.botpy.vosp.common.util.SpringUtils;
import lombok.extern.slf4j.Slf4j;

import java.time.LocalDate;
import java.time.LocalDateTime;
import java.time.temporal.ChronoUnit;
import java.util.List;
import java.util.Map;
import java.util.TreeMap;

/**
 * 签到工具类
 * 1. offset: 此类中方法 offset 参数
 *      语义{ 偏移量 }
 *
 * 2. setBit: bool 结果取反的原因
 *      https://github.com/AgeFades/AgeFades-Note/blob/master/%E5%BC%82%E5%B8%B8%E6%89%8B%E5%86%8C/Redis%E9%97%AE%E9%A2%98%E9%9B%86%E9%94%A6.md
 *
 * 3. 参考链接
 *      https://www.cnblogs.com/liujiduo/p/10396020.html
 *
 * 4. v >> 1 << 1 == v : 判断数为奇数或偶数
 *      如: v = 53, 53 >> 1 = 26, 26 << 1 = 52，所以 v >> 1 << 1 != v，即 v为偶数，而偶数的二进制底位为 1，比如 0 的二进制为 00，1 的二进制为 01
 *
 * 5. v >>= 1 : 向右位移一位并赋值给当前变量
 *      如: v == 53, v >>= 1, 此时 v == 26
 *
 * @author DuChao
 * @date 2020/12/2 2:23 下午
 */
@Slf4j
public class SignUtil {

    private static final RedisUtil redisUtil = SpringUtils.getBean(RedisUtil.class);

    // TODO 目前以下方法均未考虑过期时间

    // TODO 判断用户是否断签的逻辑，放在每次用户查询签到情况 触发

    /**
     * 客户今日签到
     *
     * @param customerId 客户id
     * @return 是否签到成功
     */
    public static Boolean doSign(String customerId) {
        // 1. 构造客户签到 key
        String signKey = buildSignKey(customerId);

        // 2. 获取用户签到bitmap
        Boolean flag = redisUtil.hasKey(signKey);
        // 2.1 判断用户签到 bitmap 是否存在
        if (!flag) {
            // 2.1.1 用户签到 bitmap 不存在时，redis 记录客户首次签到时间
            redisUtil.set(buildFirstSignDateKey(customerId), LocalDateTimeUtil.beginOfDay(LocalDateTimeUtil.now()));
            // 2.1.2 redis 客户首次签到、偏移量为0
            return !redisUtil.setBit(signKey, 0, true);
        } else {
            // 2.2.1 用户签到 bitmap 存在，获取 offset
            long offset = getOffset(customerId);
            // 2.2.2 redis 客户签到记录、偏移量为 时间差
            return !redisUtil.setBit(signKey, offset, true);
        }
    }

    /**
     * 检查客户今日是否签到 TODO 这个方法目前看起来作用不大，setBit 返回值为1时，即用户已签到
     *
     * @param customerId 客户id
     * @return 是否签到
     */
    public static Boolean checkSign(String customerId) {
        return redisUtil.getBit(buildSignKey(customerId), LocalDate.now().getDayOfMonth() - 1);
    }

    /**
     * 获取客户签到天数和 // TODO 该方法目前看起来也没啥用
     *
     * @param customerId 客户id
     * @return 客户签到天数和
     */
    public static Long getSignCount(String customerId) {
        return redisUtil.bitCount(buildSignKey(customerId));
    }

    /**
     * 获取客户当前已连续签到天数 TODO 返回值为 0 或 7 时, 应该重新初始化签到key
     *
     * @param customerId 客户id
     * @return 客户当前已连续签到天数
     */
    public static int getContinuousSignCount(String customerId) {
        // 1. 构造客户签到 key
        String signKey = buildSignKey(customerId);

        // 2. 获取用户签到bitmap
        Boolean flag = redisUtil.hasKey(signKey);

        if (!flag) {
            // 2.1 不存在, 表示连续签到为0天
            return 0;
        } else {
            // 2.2 存在时，计算时间差{offset}，获取位数组并计算
            int result = 0;

            // 2.2.1 存在时，获取 offset { 当前时间与首次签到时间的差 }，当日为0，所以需要 +1
            long offset = getOffset(customerId) + 1;

            // 2.2.2 bitfield 根据 offset 获取签到 bitmap 中的位数组
            List<Long> list = redisUtil.bitField(buildSignKey(customerId), (int) offset, 0);
            if (CollUtil.isNotEmpty(list)) {
                long offsetValue = list.get(0) == null ? 0 : list.get(0);
                // 2.2.3 位数组以当前位向前遍历
                for (int i = 0; i < offset; i++) {
                    // 2.2.4 v >> 1 << 1 == v 时，即 v 为 偶数、则二进制低位为 0
                    if (offsetValue >> 1 << 1 == offsetValue) {
                        // 2.2.5 判断是否当天、i = 0 即当天、当天低位为0时，继续向前统计
                        if (i > 0) break;
                    } else {
                        // 2.2.6 v 为奇数，即二进制低位为 1，连续天数 ++
                        ++result;
                    }
                    offsetValue >>= 1;
                }
            }
            return result;
        }

    }

    /**
     * 获取客户签到情况 如 : { 1 : true, 2 : false, 3 : true ... }
     *
     * @param customerId 客户id
     * @return 客户签到情况
     */
    public static Map<Integer, Boolean> getSignInfo(String customerId) {
        // 1. 构造客户签到 key
        String signKey = buildSignKey(customerId);

        // 2. 准备返回容器
        Map<Integer, Boolean> signInfo = new TreeMap<>();

        // 3. 判断用户是否存在签到 bitmap
        if (redisUtil.hasKey(signKey)) {
            // 3.1 存在时，获取 offset { 当前时间与首次签到时间的差 }，当日为0，所以需要 +1
            long offset = getOffset(customerId) + 1;
            // 3.2 bitfield 根据 offset 获取签到 bitmap 中的位数组
            List<Long> list = redisUtil.bitField(buildSignKey(customerId), (int) offset, 0);
            if (CollUtil.isNotEmpty(list)) {
                // 由低位到高位，为0表示未签，为1表示已签
                long v = list.get(0) == null ? 0 : list.get(0);
                // 这里统计用户最近七天签到情况
                for (int i = 1; i <= 7; i++) {
                    signInfo.put(i, v >> 1 << 1 != v);
                    v >>= 1;
                }
            }
        }

        return signInfo;
    }

    /**
     * 获取客户当月首次签到的日期 TODO 这个方法暂时应该也没用
     *
     * @param customerId 客户id
     * @return 客户当月首次签到的日期
     */
    public static LocalDate getFirstSignDate(String customerId) {
        Long bitPos = redisUtil.bitPos(buildSignKey(customerId));
        return bitPos < 0 ? null : LocalDate.now().withDayOfMonth((int) (bitPos + 1));
    }

    /**
     * 构造签到key, 格式举例: { sign:1001 | 表示客户id为1001的客户签到数据存储的 key }
     *
     * @param customerId 客户id
     * @return 签到key
     */
    private static String buildSignKey(String customerId) {
        return String.format("sign:%s", customerId);
    }

    /**
     * 构造签到首次时间key，格式举例: { sign:first:1001 | 表示客户id为1001的客户，首次签到时间存储的key }
     *
     * @param customerId 客户id
     * @return 签到首次时间key
     */
    private static String buildFirstSignDateKey(String customerId) {
        return String.format("sign:first:%s", customerId);
    }

    /**
     * 获取今日客户签到 bitmap 偏移量
     *
     * @param customerId 客户id
     * @return 今日客户签到 bitmap 偏移量
     */
    private static long getOffset(String customerId) {
        try {
            // 用户签到 bitmap 存在时，redis 获取客户首次签到时间
            LocalDateTime firstSignDay = Convert.toLocalDateTime(redisUtil.get(buildFirstSignDateKey(customerId)));
            // 比较当前时间 与 首次签到时间差异，差异天数即为 bitmap offset
            return LocalDateTimeUtil.between(firstSignDay, LocalDateTimeUtil.beginOfDay(LocalDateTimeUtil.now()), ChronoUnit.DAYS);
        } catch (Exception e) {
            log.warn("获取今日客户签到 bitmap 偏移量异常, 已返回0, 异常为: ", e);
            return 0;
        }
    }

}

```

