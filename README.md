package com.proj3.bomberman;

import sofia.graphics.Shape;
import java.util.HashMap;
import sofia.graphics.Color;
import java.util.ArrayList;
import sofia.graphics.OvalShape;
import sofia.app.ShapeScreen;
import java.util.Iterator;
import com.proj3.bomberman.BoardCell.CellTypes;
import sofia.graphics.RectangleShape;

// -------------------------------------------------------------------------
/**
 * Creates a ShapeScreen with tile objects in a 7x11 grid. Responds to touch
 * events and displays a new level when appropriate.
 * @author crc1559
 * @author aaronk1
 * @author zhinson
 * @version Nov 13, 2013
 */
public class BomberManScreen extends ShapeScreen
{

    private Tile[][] grid;
    private Tile tempTile;
    private Board board;
    private BomberMan bomberman;
    private ArrayList<Enemy> enemies;
    private ArrayList<OvalShape> cpus;
    private ArrayList<Bomb> bombs;
    private ArrayList<RectangleShape> expImages;
    private HashMap<EnemyBomb, OvalShape> bombImages;
    private OvalShape player;
    private OvalShape playerBomb;
    private int numEnemies = 1;
    private Iterator<Bomb> iter;
    private Iterator<RectangleShape> rectIter;


    public void initialize()
    {
        grid = new Tile[7][11];
        board = new Board();
        float cellWidth = getWidth()/7;
        float cellHeight = getHeight()/11;
        for (int i = 0; i < 7; i++)
        {
            for (int k = 0; k < 11; k++)
            {
                Location loc = new Location(i, k);
                Tile tile =
                    new Tile(i * cellWidth, k * cellHeight, (i + 1) * cellWidth,
                        (k + 1) * cellHeight, loc);
                add(tile);
                grid[i][k] = tile;
            }
        }

        setTileColors();

        //add bomberman (the player) to the screen
        bomberman = new BomberMan();
        tempTile = grid[0][0];
        player = new OvalShape(tempTile.getCenterPointX(),
            tempTile.getCenterPointY(), 20);
        player.setFillColor(Color.blue);
        add(player);

        //add the enemies, currently only supports up to 4, will get
        //errors otherwise
        enemies = new ArrayList<Enemy>();
        cpus = new ArrayList<OvalShape>();
        for (int i = 0; i < numEnemies; i++)
        {
            Enemy tempEnemy = new Enemy(new Location(3, (i + 1) * 2), board);
            tempTile = grid[3][(i + 1) * 2];
            enemies.add(tempEnemy);
            cpus.add(new OvalShape(tempTile.getCenterPointX(),
                        tempTile.getCenterPointY(), 20));
            cpus.get(i).setFillColor(Color.red);
            add(cpus.get(i));
        }
        bombs = new ArrayList<Bomb>();
        bombImages = new HashMap<EnemyBomb, OvalShape>();
        expImages = new ArrayList<RectangleShape>();
    }

    /**
     * is called when the player presses the screen. Handles all movement and
     * actions of the visual game.
     * @param x coordinate of pixel
     * @param y coordinate of pixel
     */
    public void onTouchDown(float x, float y)
    {
        removeExplosionImages();
        if (bomberman != null)
        {
            boolean occupied = false;
            Tile tile =
                getShapes().locatedAt(x, y).withClass(Tile.class).front();
            //Step 1: Move bomberman. His action comes first.
            Location locbomberman = bomberman.getCoord();
            int tilex = tile.getCoord().x();
            int tiley = tile.getCoord().y();
            int bomberx = locbomberman.x();
            int bombery = locbomberman.y();
            //prevents movement onto enemies
            for (Enemy enem : enemies)
            {
                if (enem.getLocation().x() == tilex
                    && enem.getLocation().y() == tiley)
                {
                    occupied = true;
                    //cut out of the for loop early if the location is occupied
                    break;
                }
            }
            if (board.getCell(tile.getCoord()) == CellTypes.PATH && !occupied)
            {
                if (tilex == bomberx && tiley == bombery + 1)
                {
                    bomberman.moveDown();
                    player.setPosition(tile.getCenterPointX(),
                        tile.getCenterPointY());
                }
                else if (tilex == bomberx && tiley == bombery - 1)
                {
                    bomberman.moveUp();
                    player.setPosition(tile.getCenterPointX(),
                        tile.getCenterPointY());
                }
                else if (tilex == bomberx + 1 && tiley == bombery)
                {
                    bomberman.moveRight();
                    player.setPosition(tile.getCenterPointX(),
                        tile.getCenterPointY());
                }
                else if (tilex == bomberx - 1 && tiley == bombery)
                {
                    bomberman.moveLeft();
                    player.setPosition(tile.getCenterPointX(),
                        tile.getCenterPointY());

                }
                else if (tilex == bomberx && tiley == bombery)
                {
                    if (bombs.size() == 0)
                    {
                        addAlliedBomb(tile);
                    }
                    else
                    {
                        for (Bomb bomb : bombs)
                        {
                            if (bomb instanceof AlliedBomb)
                            {
                                //bomb has already been placed, not a legal move
                                return;
                            }
                        }
                        addAlliedBomb(tile);
                    }
                }
                else
                {
                    //no action should be performed on the board since it was not
                    //a legal move
                    return;
                }
            }
            else
            {
                //not a legal move. No action performed
                return;
            }

            //Step 2: Have the enemies act
            for (Enemy enem : enemies)
            {
                //int i = 0;
                Location oldLoc = enem.getLocation();
                Location loc = enem.action();
                tempTile = grid[loc.x()][loc.y()];
                cpus.get(enemies.indexOf(enem)).setPosition(tempTile.getCenterPointX(),
                        tempTile.getCenterPointY());
                //i++;
                if (oldLoc.equals(loc))
                {
                    Tile enemTile = grid[loc.x()][loc.y()];
                    OvalShape ov = new OvalShape(enemTile.getCenterPointX(),
                        enemTile.getCenterPointY(), 30);
                    ov.setFillColor(Color.orange);
                    add(ov);
                    EnemyBomb bo = new EnemyBomb(enemTile.getCoord().x(),
                        enemTile.getCoord().y(), 4, 1);
                    bombs.add(bo);
                    //ensures that for every Enemybomb, there is a singular
                    //associated OvalShape
                    bombImages.put(bo, ov);

                }
            }

            //Step 3: Bombs decrease in time or go off

            checkForExplosion();

            //Step 4: update the board for the enemies assuming the game wasn't lost
            if (!enemies.isEmpty())
            {
                for (Enemy enem : enemies)
                {
                    enem.updateBoard(board);
                }
            }
            else
            {
                startNewLevel();
            }
        }
    }


    /**
     * designed to check to see if a bomb should explode
     */
    private void checkForExplosion()
    {
        iter = bombs.iterator();
        while (iter.hasNext())
        {
            Bomb bomb = iter.next();
            bomb.countdown(board);
            if (bomb instanceof AlliedBomb)
            {
                if (((AlliedBomb)bomb).hasExploded())
                {
                    setTileColors();
                    int bombx = ((AlliedBomb)bomb).getX();
                    int bomby = ((AlliedBomb)bomb).getY();
                    addExplosionImages(((AlliedBomb)bomb).getX(), ((AlliedBomb)bomb).getY());

                    //kill any enemies in blast radius
                    Iterator<Enemy> enemIter = enemies.iterator();
                    while (enemIter.hasNext())
                    {
                        Enemy enem = enemIter.next();
                        OvalShape ov;
                        Location eLoc = enem.getLocation();
                        if (eLoc.equals(new Location(bombx, bomby)))
                        {
                            ov = cpus.get(enemies.indexOf(enem));
                            cpus.remove(ov);
                            remove(ov);
                            enemIter.remove();
                        }
                        else if (eLoc.equals(new Location(bombx, bomby + 1)))
                        {
                            ov = cpus.get(enemies.indexOf(enem));
                            cpus.remove(ov);
                            remove(ov);
                            enemIter.remove();
                        }
                        else if (eLoc.equals(new Location(bombx, bomby - 1)))
                        {
                            ov = cpus.get(enemies.indexOf(enem));
                            cpus.remove(ov);
                            remove(ov);
                            enemIter.remove();
                        }
                        else if (eLoc.equals(new Location(bombx + 1, bomby)))
                        {
                            ov = cpus.get(enemies.indexOf(enem));
                            cpus.remove(ov);
                            remove(ov);
                            enemIter.remove();
                        }
                        else if (eLoc.equals(new Location(bombx + 1, bomby)))
                        {
                            ov = cpus.get(enemies.indexOf(enem));
                            cpus.remove(ov);
                            remove(ov);
                            enemIter.remove();
                        }
                    }

                    //kill bomberman if in blast radius
                    Location eLoc = bomberman.getCoord();
                    if (eLoc.equals(new Location(bombx, bomby)))
                    {
                        remove(player);
                        bomberman = null;

                    }
                    else if (eLoc.equals(new Location(bombx, bomby + 1)))
                    {
                        remove(player);
                        bomberman = null;
                    }
                    else if (eLoc.equals(new Location(bombx, bomby - 1)))
                    {
                        remove(player);
                        bomberman = null;
                    }
                    else if (eLoc.equals(new Location(bombx + 1, bomby)))
                    {
                        remove(player);
                        bomberman = null;

                    }
                    else if (eLoc.equals(new Location(bombx + 1, bomby)))
                    {
                        remove(player);
                        bomberman = null;
                    }
                    iter.remove();
                    remove(playerBomb);
                }
            }
            else
            {
                setTileColors();
                if (((EnemyBomb)bomb).hasExploded())
                {
                    addExplosionImages(((EnemyBomb)bomb).getX(), ((EnemyBomb)bomb).getY());
                    int bombx = ((EnemyBomb)bomb).getX();
                    int bomby = ((EnemyBomb)bomb).getY();
                    //enemy bombs don't kill other enemies, just bomberman
                    Location eLoc = bomberman.getCoord();
                    if (eLoc.equals(new Location(bombx, bomby)))
                    {
                        remove(player);
                        bomberman = null;
                    }
                    else if (eLoc.equals(new Location(bombx, bomby + 1)))
                    {
                        remove(player);
                        bomberman = null;
                    }
                    else if (eLoc.equals(new Location(bombx, bomby - 1)))
                    {
                        remove(player);
                        bomberman = null;
                    }
                    else if (eLoc.equals(new Location(bombx + 1, bomby)))
                    {
                        remove(player);
                        bomberman = null;
                    }
                    else if (eLoc.equals(new Location(bombx + 1, bomby)))
                    {
                        remove(player);
                        bomberman = null;
                    }
                    OvalShape ov = bombImages.get(bomb);
                    bombImages.remove(bomb);
                    iter.remove();
                    remove(ov);
                }
            }
        }
    }

    /**
     * improve some of the coloring, adds a white border to each tile for
     *  visual clarity and sets fill colors acording to CellType
     *  NOTE: setColor sets the border color
     *  NOTE: setFillColor sets the color of the tile minus the border
     *  if setFillColor is called without setColor, the border will be the
     *  color of the fill
     */
    private void setTileColors()
    {
        for (int i = 0; i < 7; i++)
        {
            for (int j = 0; j < 11; j++)
            {
                tempTile = grid[i][j];
                tempTile.setColor(Color.white);
                if (board.getCell(tempTile.getCoord()) == CellTypes.DEBRIS)
                {
                    tempTile.setFillColor(Color.lightSlateGray);
                }
                else if ((board.getCell(tempTile.getCoord()) == CellTypes.WALL))
                {
                    tempTile.setFillColor(Color.darkSlateGray);
                }
                else if ((board.getCell(tempTile.getCoord()) == CellTypes.PATH))
                {
                    tempTile.setFillColor(Color.rgb(0, 160, 0));
                }
            }
        }
    }

    private void addAlliedBomb(Tile tile)
    {
        playerBomb = new OvalShape(tile.getCenterPointX(),
            tile.getCenterPointY(), 30);
        playerBomb.setFillColor(Color.orange);
        add(playerBomb);
        bombs.add(new AlliedBomb(tile.getCoord().x(), tile.getCoord().y(),
            bomberman.getTime() + 1, 1));
    }

    private void startNewLevel()
    {
        if (numEnemies < 4)
        {
            numEnemies++;
        }
        else
        {
            numEnemies = 1;
        }
        initialize();
    }

    private void addExplosionImages(int x, int y)
    {
        System.out.println(x + " " + y);
        tempTile = grid[x][y];
        float top = tempTile.getHeight() * y;
        float bottom = tempTile.getHeight() * (y + 1);
        float left = tempTile.getWidth() * x;
        float right = tempTile.getWidth() * (y + 1);
        RectangleShape rect = new RectangleShape(left, top, right, bottom);
        rect.setFillColor(Color.purple);
        expImages.add(rect);
        add(rect);
        if (x > 0)
        {
            System.out.println("reached 1");
            tempTile = grid[x - 1][y];
            top = tempTile.getHeight() * y;
            bottom = tempTile.getHeight() * (y + 1);
            left = tempTile.getWidth() * (x - 1);
            right = tempTile.getWidth() * (x);
            rect = new RectangleShape(left, top, right, bottom);
            rect.setFillColor(Color.purple);
            expImages.add(rect);
            add(rect);
            System.out.println(top + " " + bottom + " " + right + " " + " " + left);
        }
        if (x < 6)
        {
            System.out.println("reached 2");
            tempTile = grid[x + 1][y];
            top = tempTile.getHeight() * y;
            bottom = tempTile.getHeight() * (y + 1);
            left = tempTile.getWidth() * (x + 1);
            right = tempTile.getWidth() * (x + 2);
            rect = new RectangleShape(left, top, right, bottom);
            rect.setFillColor(Color.purple);
            expImages.add(rect);
            add(rect);
            System.out.println(top + " " + bottom + " " + right + " " + " " + left);
        }
        if (y > 0)
        {
            System.out.println("reached 3");
            tempTile = grid[x][y - 1];
            top = tempTile.getHeight() * (y - 1);
            bottom = tempTile.getHeight() * (y);
            left = tempTile.getWidth() * x;
            right = tempTile.getWidth() * (x + 1);
            rect = new RectangleShape(left, top, right, bottom);
            rect.setFillColor(Color.purple);
            expImages.add(rect);
            add(rect);
            System.out.println(top + " " + bottom + " " + right + " " + " " + left);
        }
        if (y < 10)
        {
            System.out.println("reached 4");
            tempTile = grid[x][y + 1];
            top = tempTile.getHeight() * (y + 1);
            bottom = tempTile.getHeight() * (y + 2);
            left = tempTile.getWidth() * x;
            right = tempTile.getWidth() * (x + 1);
            rect = new RectangleShape(left, top, right, bottom);
            rect.setFillColor(Color.purple);
            expImages.add(rect);
            add(rect);
            System.out.println(top + " " + bottom + " " + right + " " + " " + left);
        }
    }

    private void removeExplosionImages()
    {
        for (RectangleShape rect : expImages)
        {
            remove(rect);
        }
        expImages.clear();
    }
}
