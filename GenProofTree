
package com.hbtc.proof;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.StandardOpenOption;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.util.ArrayList;
import java.util.List;
import java.util.Objects;
import java.util.concurrent.atomic.AtomicLong;
import java.util.stream.Collectors;

/**
 * HBTC Proof of Reserve
 *
 * see https://hbtc.zendesk.com/hc/zh-cn/articles/360046287754
 *
 */
public class GenProofTree {

    public static final List<String> TOKENS = List.of("btc", "eth", "usdt");
    public static final String BALANCE_LIST_FILE = "/src/main/resources/proof_reserve_";
    public static final String PROOF_TREE_DUMP_FILE = "/src/main/resources/tree_proof_";
    public static final String PROOF_TREE_DUMP_FILE_FULL = "/src/main/resources/tree_proof_full_";
    public static final int INDEX_START = 1000_000;
    public static final int HEX_LEN = 16;
    public static MessageDigest md;

    public static void main(String[] args) throws IOException, NoSuchAlgorithmException {
        md = MessageDigest.getInstance("SHA-256");

        for (String token : TOKENS) {
            generateProof(token, System.getProperty("user.dir") + BALANCE_LIST_FILE + token + ".csv",
                    System.getProperty("user.dir") + PROOF_TREE_DUMP_FILE + token + ".csv",
                    System.getProperty("user.dir") + PROOF_TREE_DUMP_FILE_FULL + token + ".csv");
        }

        // verifyProof(System.getProperty("user.dir") + PROOF_TREE_DUMP_FILE_FULL);
    }

    /**
     * TODO 目前版本仅支持验证 csv 格式，json 格式稍后提供
     */
    public static void verifyProof(String proofDumpFile) throws IOException {
        long start = System.currentTimeMillis();
        System.out.println("verifyProof start at " + start);
        final AtomicLong lineCnt = new AtomicLong(0);

        Files.lines(Path.of(proofDumpFile))
                .forEach(line -> {
                    lineCnt.incrementAndGet();
                    boolean succ = true;
                    // todo: parse json file
                });

        long end = System.currentTimeMillis();
        System.out.println("verifyProof finish at " + end);
        System.out.println("total user cnt: " + lineCnt.get());
        System.out.println("compute speed (tps): " + lineCnt.get() * 1000 / (end - start));
    }

    public static void generateProof(String token, String balanceListFile, String proofDumpFile, String proofDumpFullFile)
            throws IOException, NoSuchAlgorithmException {
        long start = System.currentTimeMillis();
        System.out.println(token + " generateProof start at " + start);

        List<UserBalance> balances = readBalances(balanceListFile);

        ProofTree tree = new ProofTree(token, 1586707200000L);
        tree.initTreeWithUserBalances(balances);
        tree.dumpToFile(proofDumpFile);
        tree.dumpToFullFile(proofDumpFullFile);

        long end = System.currentTimeMillis();
        System.out.println(token + " generateProof finish at " + end);
        System.out.println("total user cnt: " + balances.size());
        System.out.println("compute speed (tps): " + balances.size() * 1000 / (end - start));
    }

    private static List<UserBalance> readBalances(String balanceListFile) throws IOException {
        return Files.lines(Path.of(balanceListFile))
                .map(UserBalance::fromCsvLine)
                .dropWhile(Objects::isNull)
                .collect(Collectors.toList());
    }

    private static String convertBytesToHexWithLength(byte[] bytes, int length) {
        StringBuilder result = new StringBuilder();
        for (byte temp : bytes) {
            // bytes widen to int, need mask, prevent sign extension
            int decimal = (int) temp & 0xff;
            String hex = Integer.toHexString(decimal);
            result.append(hex);
        }
        return result.toString().substring(0, length);
    }


    static class UserBalance {
        Long userId;
        Long nonce;
        // amount = real_amount * 10^8, as int64
        Long amount;

        static UserBalance fromCsvLine(String csvLine) {
            UserBalance balance = new UserBalance();
            String[] parts = csvLine.split(",");
            if (parts.length != 4) {
                return null;
            }
            try {
                balance.userId = Long.parseLong(parts[1]);
                balance.nonce = Long.parseLong(parts[2]);
                balance.amount = Long.parseLong(parts[3]);

                return balance;
            } catch (NumberFormatException nfe) {
                return null;
            } catch (Exception e) {
                return null;
            }
        }

        @Override
        public String toString() {
            return userId + "|" + nonce + "|" + amount;
        }

        /**
         * sha256(str(user_id) + sprintf("%06d", nonce) + sprintf("%016lld", balance))
         *
         * @return
         */
        public byte[] toBytes() {
            return (userId + "|" + nonce + "|" + amount).getBytes();
        }
    }

    enum TreeNodeType {
        SELF,
        USER,
        ROOT,
        NODE,
        RELEVANT
    }

    static class TreeNode {
        int index;
        int level;
        long amountSum;
        byte[] hash;
        String hexHash;

        TreeNode parent;
        UserBalance userBalance;
        TreeNodeType type = TreeNodeType.NODE;

        public TreeNode(int index, int level, long amountSum, byte[] hash, String hexHash) {
            this.index = index;
            this.level = level;
            this.amountSum = amountSum;
            this.hash = hash;
            this.hexHash = hexHash;
        }

        public void setParent(TreeNode parent) {
            this.parent = parent;
        }

        public void setUserBalance(UserBalance userBalance) {
            this.userBalance = userBalance;
        }

        /**
         * this is a temp field, will change, just for toJson
         *
         * @param type
         */
        public void setType(TreeNodeType type) {
            this.type = type;
        }

        /**
         * duplicate a new TreeNode, for padding
         *
         * @return
         */
        public TreeNode dup() {
            TreeNode node = new TreeNode(index + 1, level, 0, hash, hexHash);
            return node;
        }

        @Override
        public String toString() {
            return "\n" + toCsvString();
        }

        public String toJsonString(String childrenJson) {
            return "{\"level\":" + level
                    + ", \"index\":" + index
                    + ", \"type\":\"" + type.name()
                    + "\", \"hash\":\"" + hexHash
                    + "\", \"amount\":" + amountSum
                    + ((childrenJson == null || childrenJson.isBlank()) ? "" : (", \"children\":[" + childrenJson + "]"))
                    + "}";
        }

        public String toCsvString() {
            return level + "," + index + "," + hexHash + "," + amountSum;
        }

        public static TreeNode fromCsvString(String csvString) {
            String[] parts = csvString.split(",");
            if (parts.length != 4) {
                return null;
            }
            return new TreeNode(Integer.parseInt(parts[1].trim()),
                    Integer.parseInt(parts[0].trim()),
                    Long.parseLong(parts[3].trim()),
                    null,
                    parts[2]);
        }
    }

    static class ProofTree {
        List<TreeNode> nodes;
        String token;
        long created;
        int level = 0;
        int nextIndex = 0;

        public ProofTree(String token, long created) {
            this.token = token;
            this.created = created;
            nodes = new ArrayList<>();
            level = 0;
        }

        void addTreeNode(TreeNode node) {
            nodes.add(node);
        }

        /**
         * sha256(_8bytes(left.sum + right.sum) + _8bytes(left.hash) + _8bytes(right.hash))
         *
         * @param leftNode
         * @param rightNode
         * @param index
         * @param level
         */
        void computeTreeNode(TreeNode leftNode, TreeNode rightNode, int index, int level) {
            long amount = leftNode.amountSum + rightNode.amountSum;
            byte[] hash = md.digest((amount + "|" + leftNode.hexHash + "|" + rightNode.hexHash).getBytes());
            String hexHash = convertBytesToHexWithLength(hash, HEX_LEN);

            TreeNode node = new TreeNode(index, level, amount, hash, hexHash);
            nodes.add(node);

            leftNode.setParent(node);
            rightNode.setParent(node);
        }

        void initTreeWithUserBalances(List<UserBalance> balances) {
            if (nodes.size() > 0 || level > 0) {
                throw new IllegalArgumentException("can not add user balance to an exist tree");
            }
            // padding the balance to odd
            if (balances.size() % 2 == 1) {
                balances.add(UserBalance.fromCsvLine("6002,0,0,0"));
            }

            for (int i = 0; i < balances.size(); i++) {
                UserBalance balance = balances.get(i);
                nextIndex = i;
                byte[] hash = md.digest(balance.toBytes());
                String hexHash = convertBytesToHexWithLength(hash, HEX_LEN);
                TreeNode node = new TreeNode(nextIndex, level, balance.amount, hash, hexHash);
                node.setUserBalance(balance);
                addTreeNode(node);
            }

            computerTree(0, balances.size() - 1, level + 1);
        }

        private void computerTree(int start, int end, int newLevel) {
            level = newLevel;
            // padding node to odd number
            if ((end - start + 1) % 2 == 1) {
                TreeNode lastNode = nodes.get(end);
                TreeNode paddingNode = lastNode.dup();
                addTreeNode(paddingNode);
                end += 1;
            }

            int newStart = end + 1;
            int newEnd = end;

            for (int i = start; i <= end; i++) {
                newEnd += 1;
                nextIndex = newEnd;
                TreeNode leftNode = nodes.get(i);
                TreeNode rightNode = nodes.get(i + 1);
                computeTreeNode(leftNode, rightNode, nextIndex, level);
                i = i + 1;
            }

            if (newEnd - newStart >= 1) {
                computerTree(newStart, newEnd, level + 1);
            }
        }

        public void dumpToFile(String filePath) throws IOException {
            Files.write(Path.of(filePath),
                    nodes.stream()
                            .map(TreeNode::toCsvString)
                            .collect(Collectors.toList()),
                    StandardOpenOption.TRUNCATE_EXISTING, StandardOpenOption.CREATE
            );
        }

        public void dumpToFullFile(String filePath) throws IOException {
            Files.write(Path.of(filePath),
                    nodes.stream()
                            .map(node -> {
                                if (node.level > 0) {
                                    return null;
                                }

                                long autoIncrementId = 1000_000L;
                                if (token.equals("eth")) {
                                    autoIncrementId = 2000_000L;
                                } else if (token.equals("usdt")) {
                                    autoIncrementId = 3000_000L;
                                }

                                ArrayList<TreeNode> pathNodes = new ArrayList<>();
                                TreeNode self = node;
                                self.setType(TreeNodeType.SELF);
                                pathNodes.add(self);

                                int userIndex = self.index;
                                if (userIndex % 2 == 0) {
                                    userIndex += 1;
                                } else {
                                    userIndex -= 1;
                                }
                                TreeNode user = nodes.get(userIndex);
                                user.setType(TreeNodeType.USER);
                                pathNodes.add(user);

                                StringBuilder json = new StringBuilder();
                                json.append(self.toJsonString(null));
                                json.append(",");
                                json.append(user.toJsonString(null));
                                String childrenJson = json.toString();

                                TreeNode tn = node.parent;

                                while (tn != null) {
                                    // relevant
                                    TreeNode relevant = nodes.get(tn.index);
                                    relevant.setType(TreeNodeType.RELEVANT);
                                    pathNodes.add(relevant);

                                    json = new StringBuilder();

                                    // check if root
                                    if (tn.parent == null) {
                                        relevant.setType(TreeNodeType.ROOT);
                                        json.append(relevant.toJsonString(childrenJson));
                                        break;
                                    }

                                    json.append(relevant.toJsonString(childrenJson));
                                    json.append(",");

                                    int nodeIndex = relevant.index;
                                    if (nodeIndex % 2 == 0) {
                                        nodeIndex += 1;
                                    } else {
                                        nodeIndex -= 1;
                                    }
                                    TreeNode tmpNode = nodes.get(nodeIndex);
                                    tmpNode.setType(TreeNodeType.NODE);
                                    pathNodes.add(tmpNode);

                                    json.append(tmpNode.toJsonString(null));
                                    childrenJson = json.toString();

                                    // parent
                                    tn = tn.parent;
                                }

                                StringBuilder sb = new StringBuilder();
                                UserBalance balance = node.userBalance;

                                // id, org_id, user_id, token_id, nonce, amount, proof_json, created
                                // for mysql load_data_in_file
                                sb.append(autoIncrementId + node.index);
                                sb.append("|").append("6002");
                                sb.append("|").append(balance.userId);
                                sb.append("|").append(token);
                                sb.append("|").append(balance.nonce);
                                sb.append("|").append(balance.amount);

                                sb.append("|").append(json.toString());
                                sb.append("|").append(created);

                                return sb.toString();
                            })
                            .filter(Objects::nonNull)
                            .collect(Collectors.toList()),
                    StandardOpenOption.TRUNCATE_EXISTING, StandardOpenOption.CREATE
            );
        }

        @Override
        public String toString() {
            return nodes.toString();
        }
    }

}
