public class WinChecker {

    public static boolean isWinningLine(
            Piece p1,
            Piece p2,
            Piece p3,
            Piece p4) {

        if (p1 == null ||
            p2 == null ||
            p3 == null ||
            p4 == null) {

            return false;
        }

        int a = p1.getValue();
        int b = p2.getValue();
        int c = p3.getValue();
        int d = p4.getValue();

        // caratteristiche che valgono 1 in tutti
        int commonOnes = a & b & c & d;

        // caratteristiche che valgono 0 in tutti
        int commonZeros =
                (~a & ~b & ~c & ~d) & 0b1111;

        return commonOnes != 0 || commonZeros != 0;
    }
}
